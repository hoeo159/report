
#### Introduction

#### Related Work

#### Glare Spread Function

 GSF????
 왜 2D extension을 할 때에는 PSF가 우함수 꼴이니까 cosine 성분이 들어가는 것은 알았는데 sine이 들어가는가?? general form을 위해서?
 $$
 g_{n,m}(x,y)=a_{n,m}c_{n}(x)c_{m}(y)+b_{n,m}s_{n}(x)s_{m}(y)
 $$
 $$
 g_{n,m}(x-\alpha, y-\beta)=a_{n,m}c_{n}(x-\alpha)c_{m}(y-\beta)+b_{n,m}s_{n}(x-\alpha)s_{m}(y-\beta)
$$

ringing artifact :  fourier로 approximation할 때, discontinuities에서 sinusoidal한 느낌 때문에 over/undershooting이 일어나는 현상

$\sigma$-approximation은 fourier series(summation)에 sinc function을 곱하면서 ringing의 원인이 되는 Gibbs phenomenon을 줄이는 것

Dirichlet boundary condition : boundary condition은 우리가 주목할 대상을 위해 경계를 bound하는 것, Dirichlet은 function이 domain의 경계를 따라서 수행해야하는 값을 지정하는 boundary condition. domain의 경계 값을 고정해놓고 쓸 수 있다.

즉 [a, b] x [c, d] 크기에서 $u(x, y)$를 그릴 때, $u(x,c), u(x,d)$나 $u(a,y), u(b, y)$처럼 경계에 있는 값이 0이라면 해당 rectangle의 경계는 값이 모두 0이라는 뜻이다. 본문에서는 우리가 구하는 $F(x, y)$가 Dirichlet conditions(이때 D- boundary condition과 [D- condition](https://gosamy.tistory.com/269)도 존재하는데 무엇을 의미하는 걸까??)을 만족한다면~ 으로 내용을 시작한다.

(other boundary conditions)
Neumann : 요거는 경계 값을 고정해놓지 않고 function의 미분 값이 0으로 세팅
Periodic : 말 그대로 경계 조건이 $u(a+L)=u(x)$처럼 반복되는 것, Dirichlet과 다른 점은 미분 값도 같다는 것.

(37)식 오타??? continuos