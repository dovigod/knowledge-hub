---
id: 019ecbc5-958d-7199-9793-e44bbdbe6580
title: SDF Tetrahedral Normal Calculation
topics:
  - sdf
  - ray-marching
  - normal
  - gradient
  - tetrahedral
  - shader
  - glsl
sources:
  - 019ecbc3-4e8b-75ed-8634-7f9f8cc4150d
  - 019ecbe5-6eff-7006-a913-83414a4ebbb3
created_at: '2026-06-15T14:53:04.269Z'
updated_at: '2026-06-15T15:31:00.928Z'
---
In SDF, **Tetrahedral Normal Calculation** is one of the numerical differentiation techniques used to compute the gradient (∇f).

The most intuitive method is usually Central Difference.

\[
\nabla f =
\left(
\frac{f(p+\epsilon x)-f(p-\epsilon x)}{2\epsilon},
\frac{f(p+\epsilon y)-f(p-\epsilon y)}{2\epsilon},
\frac{f(p+\epsilon z)-f(p-\epsilon z)}{2\epsilon}
\right)
\]

This requires evaluating the SDF 6 times.

---

## Tetrahedral Method

An optimization technique frequently used by [[Inigo Quilez]].

It approximates the gradient by sampling only along the 4 vertex directions of a regular tetrahedron.

```glsl
vec3 calcNormal(vec3 p)
{
    const float h = 0.0001;

    const vec2 k = vec2(1.0, -1.0);

    return normalize(
        k.xyy * map(p + k.xyy * h) +
        k.yyx * map(p + k.yyx * h) +
        k.yxy * map(p + k.yxy * h) +
        k.xxx * map(p + k.xxx * h)
    );
}
```

The directions used here are:

```text
( 1,-1,-1)
(-1,-1, 1)
(-1, 1,-1)
( 1, 1, 1)
```

---

## Why a regular tetrahedron?

Looking at these 4 vectors:

```text
v1 = ( 1,-1,-1)
v2 = (-1,-1, 1)
v3 = (-1, 1,-1)
v4 = ( 1, 1, 1)
```

They have particular properties.

### 1. Sum is zero

\[
v_1+v_2+v_3+v_4=0
\]

That is, there is no directional bias.

---

### 2. Outer product symmetry

\[
\sum_i v_i v_i^T = 4I
\]

holds.

That is, the x, y, and z axes are treated completely identically.

---

## Why does the gradient emerge?

Taylor-expanding the SDF gives

\[
f(p+h v_i)
=
f(p)
+
h\,\nabla f\cdot v_i
+
O(h^2)
\]

Multiplying by the direction vector and summing over all of them:

\[
\sum_i v_i f(p+h v_i)
\]

\[
=
f(p)\sum_i v_i
+
h\sum_i v_i(v_i^T)\nabla f
+
O(h^2)
\]

The first term vanishes because

\[
\sum_i v_i = 0
\]

The second term becomes

\[
h(4I)\nabla f
=
4h\nabla f
\]

So

\[
\sum_i v_i f(p+h v_i)
\propto
\nabla f
\]

Therefore, just by normalizing, we obtain the normal direction.

---

## Advantages

| Method | SDF evaluations |
|--------|--------|
| Central Difference | 6 |
| Tetrahedral | 4 |

We get a normal of similar quality with about **33% fewer samples**.

In [[Ray Marching]], normal calculation happens per pixel, so this makes a significant difference.

---

Intuitively,

> Central difference measures the slope along each x/y/z axis individually,
>
> while the Tetrahedral method measures slopes along the 4 directions of a regular tetrahedron and reconstructs the gradient using symmetry.

That's why in [[Shadertoy]] or IQ's SDF renderers, `calcNormal()` frequently uses 4 tetrahedral directions.

---

## Normal = Normalized Gradient of SDF

Stated simply:

> **Normal = normalized gradient of the SDF**

Mathematically:

n = ∇d / ||∇d||

where:

- d(p): SDF value
- ∇d: gradient of the distance function
- n: normal vector

---

## Central Difference (GLSL)

The most intuitive method. Move slightly around point p in each direction and measure the change in distance.

```glsl
vec3 calcNormal(vec3 p)
{
    float e = 0.001;

    return normalize(vec3(
        sdf(p + vec3(e,0,0)) - sdf(p - vec3(e,0,0)),
        sdf(p + vec3(0,e,0)) - sdf(p - vec3(0,e,0)),
        sdf(p + vec3(0,0,e)) - sdf(p - vec3(0,0,e))
    ));
}
```

This is the direct implementation of the gradient definition.

∂d/∂x ≈ (d(x+h)-d(x-h))/(2h)

---

## IQ-style 4-sample Tetrahedral

```glsl
vec3 calcNormal(vec3 p)
{
    const float h = 0.0001;
    const vec2 k = vec2(1,-1);

    return normalize(
        k.xyy * sdf(p + k.xyy*h) +
        k.yyx * sdf(p + k.yyx*h) +
        k.yxy * sdf(p + k.yxy*h) +
        k.xxx * sdf(p + k.xxx*h)
    );
}
```

Directions used:
(1,-1,-1)
(-1,-1,1)
(-1,1,-1)
(1,1,1)

These are the 4 vertex directions of a regular tetrahedron.

The gradient is approximated with only 4 samples.

---

## Why 4 samples instead of 6?

Central Difference:
+x, -x, +y, -y, +z, -z

Total 6 SDF calls.

Tetrahedral:
Only 4 directions.

Total 4 calls.

In Ray Marching, SDF evaluation is expensive, so this saves about 33%.

---

## Analytic Normal (most accurate)

Sphere:

```glsl
float sphereSDF(vec3 p)
{
    return length(p) - r;
}
```

gradient:

```glsl
normalize(p)
```

The surface normal can be computed directly.

If d(p) = ||p|| - r, then

∇d = p / ||p||

---

## When to use each method

In real [[Ray Marching]] renderers, typically:

- Simple implementation → 6-sample Central Difference
- Performance-critical → 4-sample Tetrahedral
- Accuracy-critical → Analytic Gradient

The tetrahedral version of `calcNormal` is best understood as an optimization technique that approximates the gradient with only 4 SDF evaluations.

---

## 한국어

SDF에서 **Tetrahedral Normal Calculation**은 gradient(∇f)를 구하기 위한 수치 미분 기법 중 하나입니다.

보통 가장 직관적인 방법은 중앙차분(Central Difference)입니다.

\[
\nabla f =
\left(
\frac{f(p+\epsilon x)-f(p-\epsilon x)}{2\epsilon},
\frac{f(p+\epsilon y)-f(p-\epsilon y)}{2\epsilon},
\frac{f(p+\epsilon z)-f(p-\epsilon z)}{2\epsilon}
\right)
\]

SDF를 6번 평가해야 합니다.

---

### Tetrahedral 방식

[[Inigo Quilez]]가 자주 사용하는 최적화 기법입니다.

정사면체의 4개 꼭짓점 방향으로만 샘플링해서 gradient를 근사합니다.

```glsl
vec3 calcNormal(vec3 p)
{
    const float h = 0.0001;

    const vec2 k = vec2(1.0, -1.0);

    return normalize(
        k.xyy * map(p + k.xyy * h) +
        k.yyx * map(p + k.yyx * h) +
        k.yxy * map(p + k.yxy * h) +
        k.xxx * map(p + k.xxx * h)
    );
}
```

여기서 사용하는 방향들은

```text
( 1,-1,-1)
(-1,-1, 1)
(-1, 1,-1)
( 1, 1, 1)
```

입니다.

---

### 왜 하필 정사면체인가?

이 4개 벡터를 보면:

```text
v1 = ( 1,-1,-1)
v2 = (-1,-1, 1)
v3 = (-1, 1,-1)
v4 = ( 1, 1, 1)
```

특징이 있습니다.

#### 1. 합이 0

\[
v_1+v_2+v_3+v_4=0
\]

즉 방향 편향(bias)이 없습니다.

---

#### 2. 외적 대칭성

\[
\sum_i v_i v_i^T = 4I
\]

가 성립합니다.

즉 x, y, z 축을 완전히 동일하게 취급합니다.

---

### 왜 gradient가 나오나?

SDF를 Taylor 전개하면

\[
f(p+h v_i)
=
f(p)
+
h\,\nabla f\cdot v_i
+
O(h^2)
\]

입니다.

이를 방향 벡터와 곱해서 모두 더하면

\[
\sum_i v_i f(p+h v_i)
\]

\[
=
f(p)\sum_i v_i
+
h\sum_i v_i(v_i^T)\nabla f
+
O(h^2)
\]

첫 항은

\[
\sum_i v_i = 0
\]

이라 사라집니다.

두 번째 항은

\[
h(4I)\nabla f
=
4h\nabla f
\]

가 됩니다.

즉

\[
\sum_i v_i f(p+h v_i)
\propto
\nabla f
\]

그래서 normalize만 하면 법선 방향을 얻을 수 있습니다.

---

### 장점

| 방식 | SDF 평가 횟수 |
|--------|--------|
| Central Difference | 6 |
| Tetrahedral | 4 |

약 **33% 적은 샘플링**으로 비슷한 품질의 normal을 얻습니다.

[[Ray Marching]]에서는 픽셀마다 normal 계산이 발생하므로 꽤 큰 차이가 납니다.

---

직관적으로 보면,

> 중앙차분은 x/y/z 축으로 각각 기울기를 측정하는 방식이고,
>
> Tetrahedral 방식은 정사면체 4개 방향으로 기울기를 측정한 뒤 대칭성을 이용해 gradient를 복원하는 방식입니다.

그래서 [[Shadertoy]]나 IQ의 SDF 렌더러에서 `calcNormal()`을 보면 정사면체 방향 4개를 사용하는 코드가 자주 등장합니다.

---

### 법선 = SDF의 gradient 정규화

매우 간단히 말하면:

> **법선 = SDF의 gradient(기울기)를 정규화한 것**

입니다.

수학적으로는:

n = ∇d / ||∇d||

여기서:

- d(p): SDF 값
- ∇d: 거리 함수의 gradient
- n: 법선 벡터

---

### 가장 직관적인 방법 (Central Difference)

점 p 주변을 아주 조금씩 이동해서 거리 변화량을 측정한다.

```glsl
vec3 calcNormal(vec3 p)
{
    float e = 0.001;

    return normalize(vec3(
        sdf(p + vec3(e,0,0)) - sdf(p - vec3(e,0,0)),
        sdf(p + vec3(0,e,0)) - sdf(p - vec3(0,e,0)),
        sdf(p + vec3(0,0,e)) - sdf(p - vec3(0,0,e))
    ));
}
```

gradient의 정의 그대로 구현한 것.

∂d/∂x ≈ (d(x+h)-d(x-h))/(2h)

---

### IQ(Inigo Quilez) 스타일 4샘플 정사면체(Tetrahedral) 방식

```glsl
vec3 calcNormal(vec3 p)
{
    const float h = 0.0001;
    const vec2 k = vec2(1,-1);

    return normalize(
        k.xyy * sdf(p + k.xyy*h) +
        k.yyx * sdf(p + k.yyx*h) +
        k.yxy * sdf(p + k.yxy*h) +
        k.xxx * sdf(p + k.xxx*h)
    );
}
```

사용 방향:
(1,-1,-1)
(-1,-1,1)
(-1,1,-1)
(1,1,1)

정사면체의 4개 꼭짓점 방향이다.

gradient를 4번의 샘플만으로 근사한다.

---

### 왜 6샘플 대신 4샘플을 쓰나?

Central Difference:
+x, -x, +y, -y, +z, -z

총 6회 SDF 호출.

Tetrahedral:
4개 방향만 사용.

총 4회 호출.

Ray Marching에서는 SDF 평가가 비싸므로 약 33% 절약된다.

---

### Analytic Normal (가장 정확)

구:

```glsl
float sphereSDF(vec3 p)
{
    return length(p) - r;
}
```

gradient:

```glsl
normalize(p)
```

즉 표면 법선을 직접 계산 가능.

d(p)=||p||-r 이면

∇d = p/||p||

---

### 어느 방법을 언제 쓰는가

실제 [[Ray Marching]] 렌더러에서는 보통:

- 간단 구현 → 6샘플 Central Difference
- 성능 중요 → 4샘플 Tetrahedral
- 정확도 중요 → Analytic Gradient

정사면체 버전 `calcNormal`은 gradient를 4번의 SDF 평가만으로 근사하는 최적화 기법이라고 이해하면 된다.
