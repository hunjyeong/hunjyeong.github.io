---
title: "Multiproof"
date: 2026-11-11
categories: [Mathematics]
tags: [Mathematics, ZKP]
---

## Multiproof

Dankrad Feist의 2021년 **Polynomial Commitment Scheme Multiproof**를 기준으로 한다. 단일 KZG Opening에서 출발하여 여러 Opening을 결합하는 방법, 검증 수식, 정확성 및 안전성, Verkle Tree에 적용할 때의 데이터와 비용을 구분하여 설명한다.

## 1. 왜 Multiproof가 필요한가

**Polynomial Commitment Scheme(다항식 커밋먼트 스킴, PCS)**은 다항식 전체를 공개하지 않고도 특정 점에서의 평가값을 검증하도록 하는 암호학적 기법이다. 다항식 `f(X)`에 대해 Commitment `C`를 만들면, Prover(증명자)는 `f(z) = y`라는 주장을 **Opening Proof(평가값 증명)**로 입증할 수 있다. Verifier(검증자)는 `f(X)`의 모든 계수를 몰라도 증명을 확인할 수 있다.

여러 개의 평가값을 검증해야 하면 각각에 Opening Proof를 붙일 수 있지만, 증명의 개수가 증가한다. **Multiproof(다중 증명)**는 다음 `m`개의 주장을 하나의 Opening 증명으로 모은다.

$$
f_0(z_0)=y_0,\quad f_1(z_1)=y_1,\quad\ldots,\quad f_{m-1}(z_{m-1})=y_{m-1}.
$$

여기서 `C_i = Commit(f_i)`는 각 다항식의 Commitment다. 서로 다른 `f_i`를 사용할 수 있고, 평가 지점 `z_i`가 서로 같을 수도 있다. 이 문서에서는 각 `f_i`의 차수가 `d`보다 작고, 모든 연산이 큰 소수 `p`에 대한 유한체 `𝔽_p`에서 이루어진다고 가정한다.

Multiproof의 목적은 **검증해야 할 평가값의 개수 `m`과 무관한 크기의 핵심 암호학적 증명을 만드는 것**이다. 각 `C_i`, `z_i`, `y_i`의 제공 및 재구성에 드는 데이터와 계산 비용까지 없어지는 것은 아니다.

## 2. 준비 개념: 다항식 나눗셈과 KZG Opening

### 2.1. 나머지 정리

임의의 다항식 `f(X)`와 점 `z`에 대해 다음과 같이 나눌 수 있다.

$$
f(X)-y=(X-z)q(X)+\bigl(f(z)-y\bigr).
$$

따라서

$$
f(z)=y
\quad\Longleftrightarrow\quad
(X-z)\mid (f(X)-y).
$$

즉, **`f(z)=y`가 참일 때만** 다음 몫이 나머지 없는 다항식이다.

$$
q(X)=\frac{f(X)-y}{X-z}.
$$

예를 들어 `f(X)=X²+X+3`, `z=2`라면 `f(2)=9`이므로 다음이 성립한다.

$$
\frac{f(X)-9}{X-2}=\frac{X^2+X-6}{X-2}=X+3.
$$

반대로 `y=10`이라고 주장하면 몫에 `-1/(X-2)`가 남으므로 다항식이 아니다. 이 성질이 Opening Proof의 출발점이다.

### 2.2. KZG Commitment

**Kate-Zaverucha-Goldberg Commitment(KZG Commitment)**는 다항식에 대한 Commitment와 평가값 증명을 Pairing(쌍선형 사상)으로 검증하는 방식이다. 여기서는 개념을 이해하는 데 필요한 식만 사용한다.

- `G₁`, `G₂`: 소수 `p` 차수의 타원곡선 군이며, 군 연산은 덧셈으로 표기한다.
- `[a]₁ = aG₁`, `[a]₂ = aG₂`: 스칼라 `a`를 각 군의 생성원에 곱한 점이다.
- `s`: Trusted Setup(신뢰 설정)에 사용되는 비밀값이다. 검증자는 `s` 자체가 아니라 공개된 `[s^j]₁`, `[s]₂` 등을 이용한다.
- `e: G₁ × G₂ → G_T`: Bilinear Pairing(쌍선형 페어링)이며 `e([a]₁,[b]₂)=e([1]₁,[1]₂)^{ab}`를 만족한다.

차수가 `d` 미만인 다항식 `f(X)=Σ a_j X^j`의 Commitment는 다음과 같다.

$$
C=\operatorname{Commit}(f)=[f(s)]_1
  =\sum_{j=0}^{d-1}a_j[s^j]_1.
$$

이 Commitment는 **Additive Homomorphism(덧셈 동형성)**을 만족한다.

$$
\operatorname{Commit}(af+bg)
=a\operatorname{Commit}(f)+b\operatorname{Commit}(g).
$$

이 성질 때문에 Verifier는 `f_i`를 직접 몰라도 `C_i`의 선형 결합에 대한 Commitment를 계산할 수 있다.

### 2.3. 단일 KZG Opening

`f(z)=y`를 증명할 때 Prover는 나머지 없는 몫 `q(X)`를 계산하고 다음 한 개의 군 원소를 보낸다.

$$
\pi=[q(s)]_1,\qquad
q(X)=\frac{f(X)-y}{X-z}.
$$

Verifier는 다음 등식을 확인한다.

$$
\boxed{e(C-[y]_1,[1]_2)=e(\pi,[s-z]_2).}
$$

왜냐하면 참인 Opening에 대해서는 `f(s)-y=q(s)(s-z)`이고, Pairing의 쌍선형성이 이 곱셈 관계를 확인해 주기 때문이다. 이 검증은 KZG의 계산적 Binding(커밋한 다항식을 다른 다항식으로 바꾸기 어렵다는 성질)과 Trusted Setup의 안전성에 의존한다.

## 3. 여러 Opening을 하나로 합치는 핵심 아이디어

### 3.1. 각 Opening의 몫

`m`개의 주장마다 다음 값을 생각한다.

$$
q_i(X)=\frac{f_i(X)-y_i}{X-z_i}.
$$

모든 주장이 옳다면 각각 `q_i(X)`는 다항식이다. 따라서 다항식의 합도 다항식이다. 그러나 그냥 더하면 서로 다른 잘못된 주장들의 나머지가 상쇄될 수 있다.

### 3.2. 첫 번째 Challenge `r`: 무작위 선형 결합

**Fiat-Shamir Transform(피아트-샤미르 변환)**을 사용하여 Commitment와 모든 Opening 주장을 고정한 뒤 Hash로 Challenge(도전값) `r`을 얻는다.

$$
r\leftarrow H(C_0,\ldots,C_{m-1},y_0,\ldots,y_{m-1},z_0,\ldots,z_{m-1}).
$$

실제 구현에서는 입력의 모호하지 않은 인코딩, Domain Separation(도메인 분리), 유한체 원소로의 변환이 필요하다. 그런 다음 다음 유리함수를 정의한다.

$$
\boxed{g(X)=\sum_{i=0}^{m-1}r^i\frac{f_i(X)-y_i}{X-z_i}.}
$$

모든 Opening이 참이면 `g(X)`는 다항식이며 차수는 `d-2` 이하이다. Prover는 이를 KZG로 Commit하고 `D`를 보낸다.

$$
\boxed{D=[g(s)]_1.}
$$

**주의:** `D`를 보냈다는 사실만으로 `D`가 위 식의 정확한 `g`에 대한 Commitment임이 보장되지는 않는다. 악의적인 Prover는 임의의 다른 다항식 `f_D`에 대한 `D`를 보낼 수도 있다. 따라서 다음 단계에서 `D`의 정확성을 확인해야 한다.

### 3.3. 두 번째 Challenge `t`: `D`를 고정한 뒤 임의 지점에서 검사

`D`를 받은 다음 다음 Challenge를 만든다.

$$
\boxed{t\leftarrow H(r,D).}
$$

이때 `t ≠ z_i`가 모든 `i`에 대해 성립해야 한다. 그렇지 않으면 `t-z_i`의 역원이 없어지므로, 구현은 정해진 규칙에 따라 해당 Challenge를 거절하고 다시 생성해야 한다.

`D`를 정하기 전에 `t`를 알 수 없게 하는 것이 중요하다. 잘못된 `D`가 사전에 선택된 `t` 하나에만 맞도록 조작되는 일을 막기 위해서다.

## 4. Verifier가 `g(t)`를 어떻게 확인하는가

검증하고 싶은 관계는 다음과 같다.

$$
g(t)=\sum_{i=0}^{m-1}r^i\frac{f_i(t)-y_i}{t-z_i}.
$$

하지만 Verifier는 `f_i(X)`를 모른다. 이를 해결하기 위해 오른쪽을 두 부분으로 나눈다.

$$
\underbrace{\sum_i\frac{r^if_i(t)}{t-z_i}}_{h(t)}
-
\underbrace{\sum_i\frac{r^iy_i}{t-z_i}}_{y}.
$$

여기서

$$
\boxed{h(X)=\sum_{i=0}^{m-1}\frac{r^i}{t-z_i}f_i(X)}
\qquad
\boxed{y=\sum_{i=0}^{m-1}\frac{r^iy_i}{t-z_i}.}
$$

`y`는 알려진 스칼라 `r`, `t`, `y_i`, `z_i`만으로 계산할 수 있다. 반면 `h(X)` 전체는 Verifier가 모르지만, Additive Homomorphism 덕분에 **Commitment `E`는 직접 계산**할 수 있다.

$$
\boxed{E=\sum_{i=0}^{m-1}\frac{r^i}{t-z_i}C_i=[h(s)]_1.}
$$

따라서 남은 일은 `E-D`가 표현하는 다항식을 점 `t`에서 열었을 때 값이 `y`인지 확인하는 것이다.

$$
\boxed{h(t)-f_D(t)=y.}
$$

정직한 Prover라면 `f_D(X)=g(X)`이므로 `h(t)-g(t)=y`가 된다.

## 5. Prover와 Verifier의 전체 프로토콜

### 5.1. Prover

**입력:** `f_i(X)`, `C_i`, `z_i`, `y_i` 전체. 입력 명제는 `i=0,...,m-1`에 대해 `f_i(z_i)=y_i`다.

1. 입력 Transcript를 Hash하여 `r`을 계산한다.
2. `g(X)=Σ r^i(f_i(X)-y_i)/(X-z_i)`를 만들고 `D=Commit(g)`를 전송한다.
3. `t=H(r,D)`를 계산한다. `t`는 어느 `z_i`와도 같지 않아야 한다.
4. `h(X)=Σ r^i f_i(X)/(t-z_i)`와 `y=Σ r^i y_i/(t-z_i)`를 계산한다.
5. 다음 다항식 `w(X)`에 대한 KZG Opening Proof를 계산한다.

$$
\boxed{w(X)=\frac{h(X)-g(X)-y}{X-t},\qquad \pi=[w(s)]_1.}
$$

6. 증명 `D`, `π`를 전송한다. `E`와 `y`는 Verifier가 재계산할 수 있으므로 추가로 보낼 필요가 없다.

### 5.2. Verifier

**입력:** 인증된 `C_i`, 각 Opening의 `z_i`, `y_i`, 증명 `D`, `π`, 공개 KZG 설정.

1. 동일한 Transcript로 `r`을 재계산한다.
2. `D`를 포함해 `t`를 재계산하고 `t ≠ z_i`를 확인한다.
3. 다음 두 값을 계산한다.

$$
E=\sum_{i=0}^{m-1}\frac{r^i}{t-z_i}C_i,
\qquad
y=\sum_{i=0}^{m-1}\frac{r^iy_i}{t-z_i}.
$$

4. 아래 KZG Opening 등식을 검증한다.

$$
\boxed{e(E-D-[y]_1,[1]_2)=e(\pi,[s-t]_2).}
$$

`[s-t]₂=[s]₂-t[1]₂`이므로 Verifier가 `s`를 알 필요는 없다. 식이 성립하면 정해진 안전성 가정 아래 모든 Opening을 수용한다. 이 식은 **Pairing 검증 등식 하나**지만, 직접 계산할 때 필요한 Pairing 평가가 하나라는 의미는 아니다.

## 6. 정확성 증명: 정직한 증명은 왜 통과하는가

모든 `f_i(z_i)=y_i`가 참이라고 하자. 그러면 각 `(f_i(X)-y_i)/(X-z_i)`는 다항식이고, 따라서 `g(X)`도 다항식이다.

`h(t)`와 `g(t)`의 차이를 전개하면

$$
\begin{aligned}
h(t)-g(t)
&=\sum_i r^i\frac{f_i(t)}{t-z_i}
-\sum_i r^i\frac{f_i(t)-y_i}{t-z_i}\\
&=\sum_i r^i\frac{y_i}{t-z_i}\\
&=y.
\end{aligned}
$$

따라서 `h(X)-g(X)-y`는 `X=t`에서 0이며, `X-t`로 나누어떨어진다.

$$
h(X)-g(X)-y=(X-t)w(X).
$$

양변을 `s`에 대입하여 Commitment 표기로 옮기면

$$
E-D-[y]_1=[(s-t)w(s)]_1.
$$

그러므로 Pairing의 쌍선형성에 따라

$$
\begin{aligned}
e(E-D-[y]_1,[1]_2)
&=e([(s-t)w(s)]_1,[1]_2)\\
&=e([w(s)]_1,[s-t]_2)\\
&=e(\pi,[s-t]_2).
\end{aligned}
$$

즉, **모든 Opening이 참이면 올바르게 생성한 증명은 검증에 통과한다.** 이를 Completeness(완전성)라고 한다.

## 7. 안전성: 잘못된 Opening이 왜 들키는가

**Soundness(건전성)**는 어떤 Opening이 거짓인데도 증명에 통과할 가능성이 무시할 만큼 작다는 성질이다. 여기에는 서로 다른 역할을 가진 두 Challenge가 있다.

### 7.1. `r`은 오류의 상쇄를 방지한다

각 주장의 오류를 `e_i=f_i(z_i)-y_i`로 두자. 다항식 나눗셈을 적용하면

$$
\frac{f_i(X)-y_i}{X-z_i}
=q_i(X)+\frac{e_i}{X-z_i},
$$

여기서 `q_i(X)`는 나눗셈의 다항식 부분이다. 따라서

$$
g(X)=\sum_i r^i q_i(X)+\sum_i\frac{r^i e_i}{X-z_i}.
$$

서로 다른 `z_i`에 오류가 있다면 각각의 분모를 가진 항이 남는다. 같은 `z_i=z`에 오류가 여러 개 있으면 그 극점에 남는 계수는

$$
P_z(r)=\sum_{i:z_i=z}e_i r^i
$$

이다. 모든 `e_i`가 0이면 이 값은 항상 0이다. 반대로 적어도 하나의 `e_i ≠ 0`이라면 해당 `z`에서 `P_z(r)`은 영다항식이 아니다. 차수가 최대 `m-1`인 영이 아닌 다항식이 유한체에서 0이 되는 `r`은 최대 `m-1`개다.

따라서 `r`이 균등한 무작위 유한체 원소이고 주장이 먼저 고정되었다면, 특정 오류가 우연히 상쇄될 확률은 최대 `(m-1)/p`로 제한할 수 있다(상계가 1보다 크면 자명한 상계 1을 사용). 실제 Fiat-Shamir 구현에서는 Hash의 Random Oracle 모델과 Transcript 고정 조건 등이 중요하다.

**반례:** `z_0=z_1=z`, `e_0=1`, `e_1=-1`이면 단순 합에서는 `1/(X-z)-1/(X-z)=0`으로 오류가 사라진다. 무작위 가중합을 사용하면 `(1-r)/(X-z)`가 되어 `r=1`일 때에만 상쇄된다.

### 7.2. `t`는 `D`가 올바른 다항식 관계를 따르는지 검사한다

정직하지 않은 Prover도 자신이 선택한 다항식 `f_D(X)`에 대한 Commitment `D`를 보낼 수 있다. 실제로 검사하려는 관계는

$$
f_D(t)=\sum_i r^i\frac{f_i(t)-y_i}{t-z_i}.
$$

서로 다른 평가 지점의 집합을 `Z`라 하고 `k=|Z|`라 하자. 다음 다항식으로 분모를 제거한다.

$$
Q(X)=\prod_{z\in Z}(X-z).
$$

그리고

$$
R(X)=Q(X)f_D(X)
-\sum_i r^i\bigl(f_i(X)-y_i\bigr)\frac{Q(X)}{X-z_i}
$$

를 정의한다. 각 `z_i`가 `Z`에 있으므로 `Q(X)/(X-z_i)`는 다항식이다. 따라서 `R(X)` 역시 다항식이다. 검증되는 평가값 관계는 `R(t)=0`과 동치다. 단, `Q(t)≠0`이어야 한다.

`r`이 오류 상쇄를 일으키지 않았으면 오른쪽의 유리함수는 다항식이 아니므로, 어떤 다항식 `f_D`를 선택해도 `R(X)`가 영다항식이 될 수 없다. `D`를 정한 후 무작위 `t`를 선택했다면 영이 아닌 다항식 `R`은 우연히 `R(t)=0`인 점에서만 검증을 통과할 수 있다.

차수가 `deg(f_i), deg(f_D)<d`이고 `Q`의 차수가 `k`라면

$$
\deg R\le d+k-1.
$$

따라서 이상화된 균등 무작위 `t`가 `Z`를 제외한 유한체에서 선택되는 경우, `R`이 영다항식이 아닐 때 우연히 `R(t)=0`이 될 확률은 최대 `(d+k-1)/(p-k)`다. 이는 Schwartz-Zippel Lemma(슈바르츠-지펠 보조정리)를 이용한 다항식 항등성 검사 관점의 설명이다.

이 추론은 유한체가 충분히 크고, 유효한 Commitment가 적절한 차수의 다항식에 결합되며, Prover가 `t`를 미리 선택하지 못하고, 기본 PCS Opening Proof가 안전하다는 가정에 의존한다. 두 Challenge가 단지 Hash로 생성되었다는 이유만으로 무조건적인 안전성이 확보되는 것은 아니다.

**요약:** `r`은 여러 Opening 사이의 잘못된 나머지가 서로 지워지는 것을 방지하고, `t`는 Prover가 보낸 `D`가 요구한 집계 관계를 따르는지 무작위 점에서 검사한다.

## 8. 손으로 계산하는 예시: Opening 두 개를 하나로 검증

수식의 흐름을 확인하기 위해 서로 다른 두 다항식을 사용한다. 다음 숫자는 Hash에서 나온 실제 Challenge가 아니라 **연산 확인을 위한 예시값**이다. 모든 계산은 `p`가 충분히 큰 소수인 `𝔽_p`에서 수행한다고 하자.

$$
\begin{array}{lll}
f_0(X)=X+2,&z_0=3,&y_0=5,\\
f_1(X)=X^2+1,&z_1=2,&y_1=5.
\end{array}
$$

실제로 `f_0(3)=5`, `f_1(2)=5`이므로 두 주장이 모두 참이다.

**① 몫 계산**

$$
q_0(X)=\frac{X+2-5}{X-3}=1,
\qquad
q_1(X)=\frac{X^2+1-5}{X-2}=X+2.
$$

**② 첫 번째 Challenge를 `r=2`로 둔다**

$$
g(X)=q_0(X)+2q_1(X)
=1+2(X+2)=2X+5,
\qquad
D=[2s+5]_1.
$$

**③ 두 번째 Challenge를 `t=4`로 둔다**

$$
\begin{aligned}
h(X)&=\frac{f_0(X)}{4-3}+\frac{2f_1(X)}{4-2}\\
&=(X+2)+(X^2+1)\\
&=X^2+X+3.
\end{aligned}
$$

따라서 Verifier는 `C_0`, `C_1`만으로 다음 Commitment를 계산한다.

$$
E=\frac{1}{1}C_0+\frac{2}{2}C_1=C_0+C_1.
$$

그리고 확인할 평가값은

$$
y=\frac{5}{4-3}+\frac{2\cdot5}{4-2}=10.
$$

**④ 합쳐진 다항식의 Opening 생성**

$$
h(X)-g(X)=X^2-X-2,
$$

$$
h(4)-g(4)=16-4-2=10=y.
$$

따라서

$$
\begin{aligned}
w(X)&=\frac{h(X)-g(X)-y}{X-t}\\
&=\frac{X^2-X-12}{X-4}\\
&=X+3.
\end{aligned}
$$

Prover는 `π=[s+3]₁`을 보낸다.

**⑤ Verifier의 Pairing 확인**

$$
\begin{aligned}
e(E-D-[10]_1,[1]_2)
&=e([s^2-s-12]_1,[1]_2),\\
e(\pi,[s-4]_2)
&=e([s+3]_1,[s-4]_2).
\end{aligned}
$$

`(s+3)(s-4)=s²-s-12`이므로 두 Pairing 결과가 일치한다. **Opening 두 개를 `D`, `π`라는 두 군 원소로 결합하여 검증했다.**

만약 두 번째 평가값을 거짓으로 `y_1=6`이라고 주장한다면 `q_1(X)=X+2-1/(X-2)`가 되어, 위에서 구한 다항식 `g(X)`를 그대로 사용할 수 없다. 무작위 `r`에 의한 나머지 상쇄나 무작위 `t`에서의 우연한 일치를 제외하면 검증에 실패한다.

## 9. Verkle Tree에서 무엇을 증명하는가

**Verkle Tree(버클 트리)**는 내부 노드의 자식 값을 Polynomial Commitment 등의 Vector Commitment(벡터 커밋먼트)로 결합하는 트리다. 부모의 Commitment `C_parent`는 자식 값 배열을 표현하는 다항식 `f_parent(X)`를 Commit한다. 특정 위치에 자식 `C_child`가 존재한다는 주장은 단순화하면 다음 Opening으로 표현할 수 있다.

$$
f_{\mathrm{parent}}(z_{\mathrm{child}})
=\operatorname{EncodeOrHash}(C_{\mathrm{child}}).
$$

이 식의 오른쪽은 설계된 트리 명세의 정해진 인코딩 또는 해시 방식에 따라 달라진다. 최종 Leaf에 대해서도 key/value 정보와 정의된 Leaf 인코딩 규칙으로 Opening 평가값을 만든다. 따라서 하나의 Leaf에 대한 증명에도 루트부터 Leaf까지 여러 개의 Opening이 발생한다.

여러 Leaf를 동시에 증명하면 **경로를 공유하는 노드의 Commitment는 재사용**할 수 있고, 경로에서 발생하는 Opening 모두를 하나의 Multiproof로 결합할 수 있다. 단, Verkle Tree의 정확한 Leaf 구조, 부재 증명(Non-inclusion Proof), 인코딩 방식은 사용하는 명세에 따라 달라진다.

### 9.1. 핵심 Multiproof와 전체 Witness 구분

| 구분 | 필요한 데이터 | 개수 또는 크기 |
| --- | --- | --- |
| 핵심 KZG Multiproof | 집계 Commitment `D`, Opening Proof `π` | `m`에 무관하게 군 원소 2개 |
| 경로 데이터 | 중간 노드 Commitment 및 필요한 경로 메타데이터 | 서로 다른 경로 노드 수에 의존 |
| Leaf 정보 | 검증하려는 key/value 또는 부재를 나타내는 정보 | 대상 및 트리 구조에 의존 |
| 인증된 Root | 외부에서 신뢰하거나 인증받은 Root Commitment | 별도로 확보해야 하며 보통 해당 Witness에서 재전송할 필요가 없음 |

예를 들어 **BLS12-381의 압축 `G₁` 표현이 원소당 48 B일 때**, `D`와 `π`라는 두 원소의 직렬화 크기 합계는 `96 B`다. 이 수치는 순수한 KZG Multiproof 부분에만 해당하며, 경로 Commitment, Leaf 데이터, Proof 인코딩 메타데이터 등을 제외한다. 일반적인 Verkle Witness 전체의 크기가 `96 B`라는 뜻이 아니다.

2021년 Vitalik Buterin 글의 `128~1,000 B`는 적용 스킴에 따라 달라지는 핵심 다항식 증명 크기를 대략적으로 설명한 범위다. 단일한 현재 프로토콜의 고정 직렬화 규격으로 받아들이면 안 된다.

### 9.2. `z_i`, `y_i`를 보내지 않아도 되는 조건

단순화된 Verkle Tree 증명에서는 검증자가 key와 경로를 알고 있으므로 `z_i`를 결정할 수 있고, 자식 Commitment와 Leaf 정보의 해시 또는 인코딩 규칙을 사용해 `y_i`를 계산할 수 있다. 따라서 이 값들을 별도로 전송할 필요가 없을 수 있다. 이는 **Verkle Tree의 데이터 구조를 활용한 최적화**이지 모든 PCS Multiproof에 공통된 성질은 아니다. 일반 Multiproof에서는 `z_i`, `y_i`가 검증자에게 반드시 알려져 있어야 한다.

### 9.3. KZG와 IPA의 차이

**Inner Product Argument(내적 증명, IPA)** 기반 Polynomial Commitment도 덧셈 동형성을 만족한다면 같은 집계 아이디어를 활용할 수 있다. 다만 마지막 Opening을 KZG Pairing으로 검증하지 않고 IPA Verifier로 검증한다.

- KZG: 핵심 집계 증명은 `D`와 `π`이며, `π`는 하나의 군 원소다. Pairing 기반이고 보통 Trusted Setup이 필요하다.
- IPA: 핵심 집계 증명은 `D`와 IPA Opening Proof다. IPA Proof는 보통 다항식 또는 벡터 차원 `d`에 따라 `O(log d)`개의 군 원소를 포함한다. 고정된 폭 `d`에서는 Opening 개수 `m`이 증가해도 핵심 집계 Proof 크기는 증가하지 않는다.

Go 라이브러리 `github.com/crate-crypto/go-ipa`의 `MultiProof` 구조도 `D`와 `IPA` 두 필드로 구성된다. 이것은 KZG와 IPA의 Proof 객체 구조가 동일하다는 뜻이 아니라, **집계 Commitment와 마지막 Opening Proof를 함께 전달한다**는 공통점을 보여 준다.

## 10. 계산 비용과 Prover 데이터 요구사항

### 검증

`E=Σ r^i/(t-z_i) C_i`를 계산하는 부분은 `m`개 타원곡선 점에 대한 **Multiscalar Multiplication(다중 스칼라 곱, MSM)**이다. 여기에 `r`, `t`, `y` 생성에 필요한 Hash와 유한체 연산, 그리고 KZG Pairing 또는 IPA Opening 검증이 더해진다. 따라서 핵심적인 작업량은 `m`이 커질수록 증가하지만, 핵심 증명 크기는 증가하지 않는다.

Vitalik Buterin의 2021년 글에서 수천 값의 검증을 약 `100 ms` 이내로 처리할 수 있다는 설명은 당시의 성능 추정이다. 하드웨어, 커브, 구현, Opening 개수와 트리 구조가 지정되지 않은 일반적인 성능 보장으로 사용하면 안 된다.

### 생성

Prover는 각 `q_i(X)`를 만들기 위해 `f_i(X)`를 계산할 수 있어야 한다. Verkle Tree에서는 일반적으로 **증명에 포함되는 노드의 자식 벡터 전체 또는 동일한 다항식을 재구성할 수 있는 데이터**가 필요하다. 기존의 개별 Opening Proof만 가지고 이 집계 방식을 자유롭게 수행할 수 있는 것은 아니다.

그러나 **블록체인 전체 상태가 있어야만 한다**는 단정은 지나치다. 해당 증명에 필요한 다항식이나 노드 데이터에 접근할 수 있으면 생성이 가능하다. 전체 상태를 유지하는 노드가 실제 Witness 제공에 적합한 대표적인 운영 형태라는 뜻이다.

효율적인 구현에서는 **Evaluation Form(평가값 표현)**을 사용한다. 노드 폭이 `d`이고 `ω`가 `d`차 단위근이면 `f(ω^j)`들로 다항식을 나타낼 수 있다. 이 경우 Lagrange Basis(라그랑주 기저)를 통해 Commitment를 계산하고, 다항식 몫과 외부 점 평가를 효율적으로 처리할 수 있다. `X=z_i`에서 직접 나누면 `0/0`이 생기므로 해당 점은 별도 계산이 필요하다. Dankrad Feist의 원문에는 이 최적화가 자세히 설명되어 있다.

## 11. 핵심 정리

1. 단일 Opening `f_i(z_i)=y_i`의 참 여부는 `(f_i(X)-y_i)`가 `(X-z_i)`로 정확히 나누어지는지와 연결된다.
2. Multiproof는 각 몫을 무작위 가중치 `r^i`로 합쳐 `g(X)`를 만든다. Challenge `r`은 잘못된 주장들의 나머지가 상쇄될 가능성을 낮춘다.
3. Prover가 보낸 `D`가 요구한 집계 다항식의 Commitment인지, `D`를 고정한 뒤 얻은 무작위 `t`에서 검사한다.
4. 검증자는 `E=Σ[r^i/(t-z_i)]C_i`와 `y=Σ[r^iy_i/(t-z_i)]`를 계산하고, **한 개의 KZG Opening 검증 등식**으로 모든 평가 주장에 대한 집계 증명을 확인한다.
5. KZG에서 핵심 집계 Proof는 두 군 원소 `D`, `π`로 표현되지만, Verkle Tree의 전체 Witness에는 경로 Commitment와 필요한 Leaf 정보가 별도로 포함된다.
6. 암호학적 안전성은 큰 유한체, 다항식 Commitment의 Binding, 정확한 Transcript 구성 및 무작위 Challenge를 모형화하는 가정에 의존한다.

## 참고 자료

- Dankrad Feist, [PCS multiproofs using random evaluation](https://dankradfeist.de/ethereum/2021/06/18/pcs-multiproofs.html), 2021-06-18. 집계 프로토콜의 원문.
- Dankrad Feist, [KZG polynomial commitments](https://kzg.eth.sucks/), 2020-06-16. 단일 KZG Opening과 Pairing 검증의 배경.
- Vitalik Buterin, [Verkle trees](https://vitalik.eth.limo/general/2021/06/18/verkle.html), 2021-06-18. Verkle Tree, 경로 Commitment, Multiproof 및 비용.
- Ethereum Annotated Specification, [Polynomial commitments](https://github.com/ethereum/annotated-spec/blob/master/deneb/polynomial-commitments.md). BLS12-381의 Commitment와 Proof 직렬화 크기 및 Pairing 검증.
- [go-ipa: multiproof package documentation](https://pkg.go.dev/github.com/crate-crypto/go-ipa). IPA 기반 Multiproof의 실제 자료 구조와 API.
