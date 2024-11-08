#### 1. Introduction

&emsp;Veiling glare, lens flare는 ghosts에 의한 stray lights 현상이다~ 렌즈가 있는 카메라 조리개 통이랑 센서 표면 사이의 reflection에 의해 나타나는, veiling glare와 lens flare를 구분하는 것은 ghosts의 종류에 따라 다르다. veiling glare가 좀 더 smooth edge, weaker intensity, bigger size를 담당한다. 빛이 막 기하학적인 모양으로 뚜렷하게 나타나는 것은 lens flare. 옛날에는 제거 대상이었지만 요즘은 예술학적으로나 현실성을 위해 의도적으로 표현하기도 한다~~

&emsp;두 효과를 rendering 하기 위해서 많은 시도들이 있었는데~ path tracing 같은 것은 너무 오래걸리더라~

&emsp;이를 커버하기 위해서 sparse ray tracing과 rasterization 방법이 소개 되었다. sample size를 줄여서 계산량을 줄이면서 high quailty 유지. 이후 matrix를 통해 flare mapping 방식으로 real-time rendering << 요 설명에 대해서는 아직 지식이 부족함.

&emsp;lens flare와 veiling glare를 decoupling해서 처리할 때의 장점, HDR 이미지 관련 연구에서 입증했다. main flare와 veiling glare를 분리하고 veiling glare를 parametric 형태로 rendering? 우리 논문에서 veiling glare의 parametric model을 소개, camera와 light source의 움직임에도 효율적으로 render 할 수 있도록 low-frequency fourier-series를 이용한 glare spread function의 approximation

#### 2. Related Work

#### 3. Glare Spread Function

&emsp;수식적으로는 GSF = (glare illuminance)/(total flux in the image of the source)
보통 태양 같이 거의 point light source에 대해서 눈으로 봤을 때에도 point 주변에 glare 효과가 뿌옇게 보이는 그런 느낌으로 [shift-invariant](https://pasus.tistory.com/23) GSF는 Point Spread Function의 특징을 가지고 있다. convolution으로 처리할 수 있어서 전체 image에 대한 glare를 같은 model로 linear하게 사용할 수 있다.




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