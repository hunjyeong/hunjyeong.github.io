---
title: "Multiproof"
date: 2026-11-11
last_modified_at: 2026-11-17
categories: [Cryptography]
tags: [Cryptography, PCS, KZG, Verkle Tree]
math: true
---

## What is Multiproof?

증명을 여러 개 만들어야 하는 상황을 생각해 보자. 어떤 값이 맞는지 보이는 증명이 하나 있고, 같은 종류의 증명이 수천 개 필요하다. 하나씩 만들면 증명 데이터가 수천 배로 늘어난다. **Multiproof(다중 증명)**는 이렇게 같은 종류의 주장 여러 개를 **증명 하나로 묶어서 한 번에 검증하는 방법**이다.

이 글에서 묶는 주장은 "어떤 다항식이 어떤 점에서 어떤 값을 가진다"는 주장이다. 이 주장이 무엇이고 왜 증명이 필요한지부터 보고, 그다음에 여러 개를 하나로 묶는 방법을 설명한다. 다루는 방식은 Dankrad Feist가 2021년에 제안한 [Random Evaluation(무작위 평가) 방식](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)이다.

## Polynomial Commitment and Opening

다항식 $f(X)$를 가진 사람(Prover, 증명자)이 있고, 그 다항식을 모르는 사람(Verifier, 검증자)이 있다. Prover는 다항식 전체를 보내지 않고 "$f(z)=y$다"라는 사실만 Verifier에게 납득시키고 싶다.

이를 위해 Prover는 먼저 다항식을 **Commit(커밋)**한다. Commit은 다항식 전체를 고정 크기의 값 하나로 압축하는 것이고, 그 결과를 **Commitment(커밋먼트)** $C$라고 한다.

Multiproof에 필요한 Commitment의 성질은 **Binding(구속성)**이다. $C$를 보낸 뒤에는 $f(X)$를 다른 다항식으로 바꿀 수 없다.

$C$만 보고 $f(X)$를 알 수 없다는 성질은 Hiding(은닉성)이라고 한다. 이 글의 방식은 Hiding을 필요로 하지 않고, 보장하지도 않는다. KZG Commitment는 다항식을 대부분 숨기지만 완전히 숨기지는 않는다. 후보가 적어서 다항식을 추측할 수 있으면, 추측한 다항식을 직접 Commit해서 $C$와 비교하면 된다([KZG polynomial commitments](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html)).

Commitment를 보낸 다음, Prover는 특정 점에서의 값을 공개한다. 이것을 **Opening(열기)**이라고 한다. 점 $z$에서 연다는 것은 "$f(z)=y$다"라는 값 $y$와, 그 값이 $C$에 묶인 다항식에서 나온 것임을 보이는 증명을 함께 보내는 것이다. 이 증명을 **Opening Proof(열기 증명)**라고 한다. Verifier는 $C$, $z$, $y$, Opening Proof만으로 주장을 확인한다. $f(X)$ 자체는 끝까지 보지 않는다.

이 구조 전체, 즉 Commit과 Opening과 그 검증 방법을 묶어 **Polynomial Commitment Scheme(다항식 커밋먼트 스킴, PCS)**이라고 한다.

이제 Multiproof의 문제를 정확히 적을 수 있다. 다항식 $m$개 $f_0, \ldots, f_{m-1}$와 각각의 Commitment $C_0, \ldots, C_{m-1}$이 있다. Prover는 다음 $m$개의 Opening을 한꺼번에 증명하고 싶다.

$$
f_0(z_0)=y_0,\quad f_1(z_1)=y_1,\quad \ldots,\quad f_{m-1}(z_{m-1})=y_{m-1}
$$

Verifier는 $C_i$, $z_i$, $y_i$를 모두 알고 있다. 목표는 Opening Proof를 $m$개 보내는 대신 **하나만 보내는 것**이다.

## Why Multiproof?

이 문제가 실제로 생기는 곳이 [Verkle Tree(버클 트리)](https://vitalik.eth.limo/general/2021/06/18/verkle.html)다. Verkle Tree는 부모 노드가 자식들을 Hash(해시) 대신 Commitment 하나로 요약하는 트리다. 자식 값 $v_0, \ldots, v_{255}$가 있으면, $f(i)=v_i$를 만족하는 다항식 $f$를 만들고 그 Commitment를 부모 노드의 값으로 삼는다. 자식이 Commitment이면 그것을 숫자로 변환한 값이 $v_i$다. 이렇게 하면 "$i$번째 자식이 $v_i$다"라는 말이 "$f(i)=v_i$다"라는 Opening이 된다.

트리에서 Leaf 하나의 값이 맞는지 확인하려면, Root부터 그 Leaf까지 내려오는 경로의 각 단계에서 "부모를 이 위치에서 열면 이 자식이 나온다"를 확인해야 한다. 깊이가 3이면 Opening이 3개다. 여러 Key를 한꺼번에 증명하면 Opening은 수백 개에서 수천 개가 된다([PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html)). [EIP-6800](https://eips.ethereum.org/EIPS/eip-6800)은 가스 한도 15M을 기준으로 블록 하나의 상태 접근을 최악의 경우 6000회로 가정한다. Opening Proof를 하나씩 보내면 그 수만큼 증명 데이터가 커진다.

Multiproof는 이 Opening들을 Opening Proof 하나로 묶는다. 다만 묶이는 것은 증명뿐이다. 경로 위의 Commitment와 읽은 값들은 그대로 보내야 한다. 이 점은 Multiproof in Verkle Tree에서 다시 본다.

## Key Idea

한 점에서의 Opening이 왜 증명 가능한지부터 보자. 출발점은 다항식에 대한 다음 사실이다.

$$
f(z)=y \quad\iff\quad f(X)-y \text{가 } (X-z)\text{로 나누어떨어진다}
$$

$f(z)=y$이면 $f(X)-y$는 $X=z$에서 0이 되므로 $(X-z)$를 인수로 가진다. 거꾸로 $(X-z)$를 인수로 가지면 $X=z$를 넣었을 때 0이므로 $f(z)=y$다. 이 사실을 Factor Theorem(인수정리)이라고 한다. "$f(X)$를 $(X-z)$로 나눈 나머지는 $f(z)$다"라는 Polynomial Remainder Theorem(다항식 나머지 정리)을 $f(X)-y$에 적용한 결과다.

예를 들어 $f(X)=X^2+X+3$이면 $f(2)=9$다. 실제로 나누어떨어진다.

$$
\frac{f(X)-9}{X-2}=\frac{(X+3)(X-2)}{X-2}=X+3
$$

반면 $f(2)=10$이라고 거짓 주장을 하면 나머지가 남는다.

$$
\frac{f(X)-10}{X-2}=X+3-\frac{1}{X-2}
$$

따라서 "$f(z)=y$다"를 증명하는 것은 **"몫 $q(X)=\dfrac{f(X)-y}{X-z}$가 다항식이다"를 증명하는 것**과 같다. KZG에서는 이 몫의 Commitment가 곧 Opening Proof다. Prover는 몫 $q(X)$의 Commitment를 보내고, Verifier는 그것이 정말 $f(X)-y$를 $(X-z)$로 나눈 몫인지 확인한다. Opening Proof의 형태와 확인 방법은 PCS마다 다르며, Final Verification with KZG에서 KZG의 방법을 본다.

이제 Opening이 $m$개다. 몫도 $m$개 생긴다.

$$
q_i(X)=\frac{f_i(X)-y_i}{X-z_i},\qquad i=0,\ldots,m-1
$$

모든 주장이 참이면 몫 $m$개가 전부 다항식이고, 그러면 **전부 더한 것도 다항식**이다. 반대로 어느 하나가 거짓이면 그 몫에는 $\dfrac{\delta}{X-z}$ 꼴의 나머지가 남는다. 그래서 Multiproof의 핵심 아이디어는 다음 한 줄이다.

> 몫 $m$개를 더해서 하나로 만들고, 그 합이 다항식임을 한 번만 증명한다.

그런데 그냥 더하면 문제가 있다. 거짓 주장이 둘 이상이면 나머지들이 서로 상쇄되어 합이 우연히 다항식이 될 수 있다. 예를 들어 $\dfrac{1}{X-2}$와 $\dfrac{-1}{X-2}$를 더하면 0이다. 이를 막기 위해 각 몫에 **Prover가 미리 알 수 없는 무작위 가중치**를 곱해서 더한다. 가중치를 모르는 상태에서 상쇄가 일어나도록 거짓 값을 맞추는 것은 사실상 불가능하다.

## Protocol

아래 계산은 소수 $p$에 대한 Finite Field(유한체) $\mathbb{F}_p$에서 한다. 나눗셈은 역원을 곱하는 연산이다. 모든 $f_i(X)$의 차수는 $d$ 미만이라고 가정한다. Commit할 수 있는 다항식도 차수 $d$ 미만으로 제한된다고 가정한다.

### Combining Quotients with $r$

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

### Checking $D$ with $t$

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

뒤쪽 $y$는 $r, t, z_i, y_i$만으로 계산할 수 있으니 Verifier가 직접 구한다. 원문은 이 값을 $g_2(t)$라고 쓴다.

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

$E$의 계산은 Multi-Exponentiation(다중 지수 연산) 한 번이고, Verifier 작업의 대부분을 차지한다.

이제 확인할 것을 다시 쓰면 $g(t)=h(t)-y$, 즉

$$
\boxed{h(t)-g(t)=y}
$$

이다. $h-g$의 Commitment는 $E-D$다. 그러므로 **$m$개의 Opening을 확인하는 문제가 "$E-D$를 $t$에서 열면 $y$가 나온다"는 Opening 하나로 바뀌었다.** Prover는 이 Opening에 대한 Opening Proof 하나를 만든다. 이것을 $\pi$라고 한다.

Prover가 보내는 것은 **$D$와 $\pi$, 두 개뿐이다.** $E$와 $y$는 Verifier가 계산한다.

$t$가 어떤 $z_i$와 같으면 분모가 0이 된다. $t$는 $\mathbb{F}_p$ 전체에서 뽑히고 서로 다른 $z_i$는 많아야 $m$개이므로 그럴 확률은 $m/p$ 이하다. 이더리움의 Verkle Tree 설계에서는 $z_i$가 0부터 255까지의 정수이고, $p$는 이 설계가 쓰는 Scalar Field의 크기인 253비트 소수이므로 무시할 수 있다([Verkle tree structure](https://blog.ethereum.org/2021/12/02/verkle-tree-structure)). 원문은 KZG를 기준으로 쓰여서 $z_i$를 $d$차 Root of Unity(단위근) $\omega$의 거듭제곱 $\omega^i$로 둔다. $t$가 $z_i$와 같아지는 경우는 원문이 다루지 않으며, 위 확률은 직접 계산한 것이다.

### Final Verification with KZG

마지막 Opening을 실제로 확인하려면 구체적인 PCS가 필요하다. 여기서는 KZG(Kate-Zaverucha-Goldberg) Commitment를 쓴다.

KZG는 Trusted Setup(신뢰 설정)에서 비밀값 $s$를 하나 뽑고, $[s^i]_1$과 $[s^i]_2$ 형태의 공개 매개변수만 남긴 뒤 $s$를 폐기한다. $[a]_1$은 Scalar(스칼라) $a$를 타원곡선 군의 생성원에 곱한 원소이고, $[a]_2$는 다른 군에서 같은 것이다. 다항식의 Commitment는 $[f(s)]_1$이다. 다항식의 계수를 공개 매개변수에 곱해서 더하면 $s$를 모르고도 계산할 수 있다.

KZG의 Opening Proof는 Key Idea에서 본 몫의 Commitment다. 마지막 Opening에 대한 몫은

$$
w(X)=\frac{h(X)-g(X)-y}{X-t}
$$

이고, Prover는 $\pi=\operatorname{Commit}(w)$를 보낸다. 정리하면

$$
D=[g(s)]_1,\qquad E=[h(s)]_1,\qquad \pi=[w(s)]_1
$$

이다. $w$의 정의에서 $h(X)-g(X)-y=(X-t)\,w(X)$이므로, $X=s$를 넣으면 $h(s)-g(s)-y=(s-t)\,w(s)$가 성립해야 한다. 좌변은 $E-D-[y]_1$에 들어 있고, 우변은 $\pi$에 $(s-t)$를 곱한 것이다. 그런데 $s$를 아무도 모르므로 곱셈을 직접 할 수 없다. 이때 쓰는 것이 Bilinear Pairing(쌍선형 페어링) $e$다. Pairing은 두 군의 원소를 받아 지수끼리 곱한 관계를 확인하게 해 준다. Verifier는 다음 등식을 확인한다.

$$
\boxed{e\big(E-D-[y]_1,\;[1]_2\big)=e\big(\pi,\;[s-t]_2\big)}
$$

$[s-t]_2$는 공개 매개변수 $[s]_2$에서 $t[1]_2$를 빼서 얻는다. 등식은 하나이지만 양변에 Pairing이 하나씩 있으므로 Pairing은 두 번 계산한다.

KZG에서 $D$와 $\pi$는 각각 군 원소 하나다. 따라서 **Opening이 몇 개든 증명 $(D,\pi)$의 크기는 같다.** BLS12-381 곡선이면 군 원소 하나가 48바이트이므로 96바이트다.

## Why It Works

**모든 주장이 참인 경우.** $g(X)$는 다항식이고, 식을 그대로 정리하면

$$
h(t)-g(t)
=\sum_i\frac{r^i f_i(t)}{t-z_i}
-\sum_i\frac{r^i\,(f_i(t)-y_i)}{t-z_i}
=\sum_i\frac{r^i y_i}{t-z_i}=y
$$

이므로 $h(X)-g(X)-y$는 $(X-t)$로 나누어떨어진다. 정직한 Prover의 $\pi$는 검증을 통과한다. 이 성질을 Completeness(완전성)라고 한다.

**거짓 주장이 섞인 경우.** $i$번째 주장의 오류를 $\delta_i=f_i(z_i)-y_i$라고 하자. Key Idea에서 본 대로 몫은 다항식 부분과 나머지로 나뉜다.

$$
\frac{f_i(X)-y_i}{X-z_i}=q_i(X)+\frac{\delta_i}{X-z_i}
$$

$g(X)$에서 같은 점 $z$에 해당하는 나머지들을 모으면 $\dfrac{P_z(r)}{X-z}$, $P_z(r)=\sum_{i:\,z_i=z}\delta_i r^i$가 된다. 거짓 주장이 있으면 $P_z$는 $r$에 대한 0이 아닌 다항식이고 차수는 $m-1$ 이하다. $r$을 뽑은 뒤 $P_z(r)=0$이 될 확률은 최대 $(m-1)/p$다. 서로 다른 점의 나머지는 분모가 다르므로 상쇄되지 않는다. 즉 $r$은 **상쇄를 막는다.**

그래도 Prover는 다항식이 아닌 $g$ 대신 아무 다항식 $\tilde g$를 Commit해 $D$로 보낼 수 있다. $\tilde g$의 차수도 $d$ 미만이다. 이때 Checking $D$ with $t$의 검증은 $g(t)=\tilde g(t)$를 확인하는 것과 같다. $g-\tilde g$는 0이 아닌 유리함수이고, 분모를 곱하면 차수 $d+m$ 이하의 0이 아닌 다항식이 된다. 무작위 $t$에서 이것이 0이 될 확률은 최대 $(d+m)/p$다. 0이 아닌 다항식이 무작위 점에서 0이 될 확률은 차수를 $p$로 나눈 값을 넘지 않는다는 사실을 Schwartz-Zippel Lemma(슈바르츠-지펠 보조정리)라고 한다. 변수가 하나일 때는 차수 $n$인 다항식의 근이 $n$개 이하라는 사실과 같다. 즉 $t$는 **$D$가 진짜 $g$의 Commitment인지 검사한다.**

원문은 이 논리를 Schwartz-Zippel Lemma로 설명할 뿐 확률을 수치로 적지 않는다. 위의 $(m-1)/p$와 $(d+m)/p$는 식에서 직접 유도한 한계다.

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

이 절은 이더리움의 Verkle Tree 제안인 Ethereum Improvement Proposal(이더리움 개선 제안, EIP) 6800의 설계를 설명한다. 이 제안은 이더리움 메인넷에 적용되지 않았다. 2026년 11월 기준 EIP-6800은 Stagnant(정체) 상태이고, 2026년 3월 6일에 consensus-specs 저장소에서 EIP-6800 사양이 [제거](https://github.com/ethereum/consensus-specs/pull/4984)됐다. Ethereum Foundation은 2026년 2월 18일 글에서 상태 저장 구조의 장기 방향을 [Binary Tree(이진 트리)와 Statelessness(무상태성)](https://blog.ethereum.org/2026/02/18/protocol-priorities-update-2026)로 적었다. Binary Tree 제안은 [EIP-7864](https://eips.ethereum.org/EIPS/eip-7864)다.

EIP-6800의 트리는 Why Multiproof?에서 본 단순한 트리와 Leaf 쪽 구조가 다르다([Verkle tree structure](https://blog.ethereum.org/2021/12/02/verkle-tree-structure)).

- Key는 32바이트다. 앞 31바이트를 Stem(스템), 마지막 1바이트를 Suffix(서픽스)라고 한다.
- 같은 Stem을 가진 값 256개는 Extension Node(확장 노드) 하나에 모인다. Extension Node의 Commitment는 $(1, \text{Stem}, C_1, C_2)$ 네 값에 대한 Commitment이고, 값들은 $C_1$과 $C_2$에 들어간다.
- 체의 크기가 253비트라서 32바이트 값은 원소 하나에 들어가지 않는다. 그래서 값은 16바이트 두 조각으로 나뉘어 들어간다. 아래쪽 조각에는 $2^{128}$을 더해 값이 0인 경우와 비어 있는 경우를 구분한다.
- 자식 Commitment는 군 원소이므로 `group_to_scalar_field` 함수로 체의 원소로 바꾼 뒤 부모의 평가값으로 쓴다.

따라서 Key 하나에 필요한 Opening 수는 경로의 깊이보다 많다. 구현체 [go-verkle](https://github.com/ethereum/go-verkle/blob/master/tree.go) 기준으로 Key 하나는 경로 위 Internal Node마다 Opening 1개, Extension Node에서 3개(위치 0, 위치 1, 그리고 위치 2 또는 3), $C_1$ 또는 $C_2$에서 2개를 만든다. 같은 Commitment를 같은 위치에서 여는 Opening은 한 번만 센다.

블록 하나를 검증할 때 보내는 데이터를 Witness(위트니스)라고 한다. 구현체 [go-verkle](https://github.com/ethereum/go-verkle/blob/master/proof_ipa.go) 기준으로 Witness는 다음으로 이루어진다.

- **State Diff:** Stem별로 묶은 Suffix, 현재 값, 새 값
- **CommitmentsByPath:** Root부터 각 Leaf까지의 연결을 확인하는 데 필요한 경로 위의 Commitment. 경로가 겹치면 한 번만
- **DepthExtensionPresent:** 각 Stem의 깊이와 Extension 상태를 담은 바이트
- **OtherStems:** 어떤 Stem이 트리에 없음을 증명할 때 쓰는 다른 Stem
- **Multiproof:** $D$와 마지막 Opening Proof

Root Commitment는 부모 블록에 이미 있는 값이므로 보내지 않는다. 평가점 $z_i$는 Key에서, 평가값 $y_i$는 자식 Commitment나 Leaf 값에서 Verifier가 직접 계산하므로 이것도 보내지 않는다.

Multiproof로 고정되는 것은 $D$와 마지막 Opening Proof뿐이다. 경로 위의 Commitment 수는 읽은 Key 수에 따라 늘어난다. 그래서 **증명 크기가 일정하다는 말과 Witness 크기가 일정하다는 말은 다르다.**

Prover 쪽 조건도 하나 있다. $g(X)$를 만들려면 각 $f_i(X)$ 전체가 필요하다. 트리로 말하면 경로 위 각 노드의 자식 값 전부(너비 256이면 256개)가 필요하고, 열려는 자식 하나만으로는 만들 수 없다. 원문도 이 방식이 다항식 전체를 가진 경우에만 효율적이고, 데이터 없이 증명들만 모아 합치는 용도에는 맞지 않는다고 적는다. 원문이 용도로 든 것은 Weak Statelessness(약한 무상태성)이며, 여기서는 [블록을 만드는 쪽만 전체 상태를 가진다](https://ethereum.org/roadmap/statelessness/).

## Beyond KZG

Protocol에서 KZG에 의존하는 부분은 마지막 Opening Proof $\pi$의 형태와 그 검증뿐이다. $D$, $E$, $y$를 만드는 과정은 Additive Homomorphism만 있으면 성립한다. 그래서 같은 방식을 Pedersen Commitment(페더슨 커밋먼트)와 [Inner Product Argument(내적 증명, IPA)](https://dankradfeist.de/ethereum/2021/07/27/inner-product-arguments.html)에도 쓸 수 있다.

EIP-6800은 KZG가 아니라 Bandersnatch 곡선 위의 Pedersen Commitment를 쓴다. Opening은 IPA로 증명한다. IPA는 EIP-6800 본문이 아니라 [Verkle tree structure](https://blog.ethereum.org/2021/12/02/verkle-tree-structure) 글과 구현체에 나온다. Trusted Setup과 Pairing이 필요 없는 대신, 마지막 Opening Proof가 군 원소 하나가 아니라 군 원소 $2\log_2 d$개와 Scalar 하나이고, 검증 비용이 $d$에 비례한다. Opening을 하나로 묶어 한 번만 검증하므로 이 비용을 모든 Opening이 나누어 부담한다.

구현체 [go-ipa](https://github.com/crate-crypto/go-ipa/blob/master/multiproof.go)의 `MultiProof`는 $D$와 IPA Proof 하나로 이루어진다. IPA Proof는 `L`, `R`, `A_scalar`로 이루어진다. $d=256$이면 `L`과 `R`이 각각 군 원소 8개다. go-verkle은 각 원소를 32바이트로 직렬화하므로 Multiproof 전체는 $32+16\times32+32=576$바이트다.

go-ipa의 Fiat-Shamir 입력 순서는 원문과 다르다. 원문은 $C_i$ 전체, $y_i$ 전체, $z_i$ 전체 순서로 Hash한다. go-ipa는 Opening마다 $C_i$, $z_i$, $y_i$ 순서로 넣어 $r$을 얻고, $D$를 넣어 $t$를 얻은 뒤, $E$를 넣고 IPA로 넘어간다. 안전성 논리는 같지만, 순서가 다르면 Challenge가 달라지므로 구현끼리 호환하려면 순서가 정확히 같아야 한다.

이 방식은 타원곡선 Commitment의 Additive Homomorphism에 의존한다. Vitalik Buterin은 양자 컴퓨터가 이 성질에 기반한 구조를 안전하지 않게 만든다고 [적었다](https://vitalik.eth.limo/general/2021/06/18/verkle.html).

## Summary

1. "$f(z)=y$다"는 "몫 $\frac{f(X)-y}{X-z}$가 다항식이다"와 같다. KZG에서는 이 몫의 Commitment가 Opening Proof다.
2. Opening이 $m$개면 몫도 $m$개다. 무작위 가중치 $r^i$를 곱해 더한 $g(X)$의 Commitment $D$를 보낸다.
3. $D$가 고정된 뒤 무작위 점 $t$를 뽑는다. Verifier는 $C_i$들을 결합해 $E$를, 공개 값으로 $y$를 계산한다.
4. $E-D$를 $t$에서 열면 $y$가 나오는지 Opening Proof $\pi$ 하나로 확인한다.

증명 $(D,\pi)$의 크기는 Opening 개수와 무관하다. 이 기법은 Zero-Knowledge(영지식)를 제공하지 않는다. $z_i$와 $y_i$는 공개된다.