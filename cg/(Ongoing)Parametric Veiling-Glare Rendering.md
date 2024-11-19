#### 1. Introduction

&ensp;Veiling glare, lens flare는 ghosts에 의한 stray lights 현상이다~ 렌즈가 있는 카메라 조리개 통이랑 센서 표면 사이의 reflection에 의해 나타나는, veiling glare와 lens flare를 구분하는 것은 ghosts의 종류에 따라 다르다. veiling glare가 좀 더 smooth edge, weaker intensity, bigger size를 담당한다. 빛이 막 기하학적인 모양으로 뚜렷하게 나타나는 것은 lens flare. 옛날에는 제거 대상이었지만 요즘은 예술학적으로나 현실성을 위해 의도적으로 표현하기도 한다

&ensp;두 효과를 rendering 하기 위해서 많은 시도들이 있었는데~ path tracing 같은 것은 너무 오래걸리더라

&ensp;이를 커버하기 위해서 sparse ray tracing과 rasterization 방법이 소개 되었다. sample size를 줄여서 계산량을 줄이면서 high quailty 유지. 이후 matrix를 통해 flare mapping 방식으로 real-time rendering << 요 설명에 대해서는 아직 지식이 부족함.

&ensp;lens flare와 veiling glare를 decoupling해서 처리할 때의 장점, HDR 이미지 관련 연구에서 입증했다. main flare와 veiling glare를 분리하고 veiling glare를 parametric 형태로 rendering? 우리 논문에서 veiling glare의 parametric model을 소개, camera와 light source의 움직임에도 효율적으로 render 할 수 있도록 low-frequency fourier-series를 이용한 glare spread function의 approximation

#### 2. Related Work

#### 3. Glare Spread Function

&ensp;수식적으로는 GSF = (glare illuminance)/(total flux in the image of the source)
&ensp;보통 태양 같이 거의 point light source에 대해서 눈으로 봤을 때에도 point 주변에 glare 효과가 뿌옇게 보이는 그런 느낌으로 [shift-invariant](https://pasus.tistory.com/23) 

PSF : 점광원에 대해서 impulse point를 퍼뜨리는 함수
Shift-invariant : 입력의 위치가 바뀌어도 spread 패턴이 동일, 영상의 어느 위치에서나 동일한 PSF를 적용할 수 있다?

GSF는 shift-invarient 하다면 PSF의 특징을 가지고 있다. convolution으로 처리할 수 있어서 전체 image에 대한 glare를 같은 model로 linear하게 사용할 수 있다.


#### 4. Model

##### 4.1 1D Fourier Series Expansion

&ensp;main idea는 glare의 PSF가 $\lambda$의 period를 가지고 있다고 가정한다. 특히 스크린 사이즈를 $\lambda$의 절반 이하로 설정하면서 화면에서 주기적인 반복이 없더라도 주기함수로 가정할 수 있다. 모든 frequency $f_{k}$에 대해 T를 동일하게 해서 다른 ghosts 집합을 하나의 Fourier Series로 만들 수 있다. sinusoidal function 기본 sine, cosine을 축약어로 나타낸다.
$$
c_{n}(x) := \cos(n\omega x), \quad\quad s_{n}(x):=\sin(n\omega x)
$$
&ensp;glare PSF는 대부분 even function이기 때문에 다음과 같이 표현할 수 있다
$$
f(x) = \sum_{n=0}^{N}a_{n}c_{n}(x)
$$
&ensp;또한 스크린 내부만을 제한해서 general form을 표현한다면, (세부 증명은 [CH](2024-11-01-Circular-Harmonics) 참고)
$$
a_{n}=\frac{\tau}{\lambda}\int _{-\lambda}^{\lambda}f(x)c_{n}(x)dx\quad \text{where, } \tau=1 \text{ if n = 0, otherwise } \tau=2.
$$
##### 4.2 2D Extension of PSFs

&ensp;2D 역시  same period T를 사용하기 때문에 동일하게 표현할 수 있다.
$$
f(x,y)=\sum_{n=0}^{N}\sum_{m=0}^{M}g_{n,m}(x,y)
$$
&ensp;여기서 $g_{n,m}(x,y)$를 다음과 같이 나타낼 수 있는데, 2D extension을 할 때에는 PSF가 우함수 꼴이니까 cosine 성분이 들어가는 것은 알았는데 sine이 들어가는가? 비대칭적 요소도 포함시키기 위해?
$$
g_{n,m}(x,y)=a_{n,m}c_{n}(x)c_{m}(y)+b_{n,m}s_{n}(x)s_{m}(y)
$$
$$
g_{n,m}(x-\alpha, y-\beta)=a_{n,m}c_{n}(x-\alpha)c_{m}(y-\beta)+b_{n,m}s_{n}(x-\alpha)s_{m}(y-\beta)
$$
##### 4.3 General Glare Spread Function

&ensp;더 general form을 위해 ghost k가 나타내는 PSF $f_{k}$와 ghost가 이동된 position vector를 $\vec{p_{k}}:=(x_{k}, y_{k})$라 했을 때 cosine과 sine으로 이루어진 4개의 항의 합으로 나타낸다. 또한 각 항들의 coefficient 역시 DFT로 표현된다. 이때 중요 파트가 $A_{n,m}$같은 항을 봤을 때 ghost k의 position vector에 관계있고 (x, y)에 독립적이기 때문에 pixel 연산 전에 pre-compute 가능하다. 즉 pixel-wise processing에서 ghost의 개수 K와는 독립적으로 approximation하는 데에 N, M이 결정하게 된다.

##### 4.4 Separabiliy

&ensp; Fourier Series 표현을 분리하여 $A_{n,m}$을 DFT 할 때 row, column 해서 계산하듯이 x, y축으로 독립적으로 하나를 fix해서 계산하고 중간에 texture 저장할 수 있다. 이를 통해 ghost의 개수 K에 dependency가 있던 시간복잡도를 해결할 수 있다.
$$
\begin{aligned}
\sum_{n=0}^{N}\sum_{m=0}^{M}A_{n,m}c_{n}(x)c_{m}(y)&=\sum_{n=0}^{N}c_{n}(x)\sum_{m=0}^{M}A_{n,m}c_{m}(y) \\
&= \sum_{n=0}^{N}c_{n}(x)A_{n,M}
\end{aligned}

$$
##### 4.5 De-ringing with $\sigma$-approximation

ringing artifact :  fourier로 approximation할 때, discontinuities에서 sinusoidal한 느낌 때문에 over/undershooting이 일어나는 현상

$\sigma$-approximation은 fourier series(summation)에 sinc function을 곱하면서 ringing의 원인이 되는 Gibbs phenomenon을 줄이는 것
##### 4.6 Positive-valued 2D Extension of PSFs

&ensp;위의 $\sigma$-approximation을 통해 위의 Fourier summation을 조정할 수 있다. 이 역시 불연속구간에서 Gibbs phenomenon이 발생하기 때문. 수식적으로 다음과 같다.
$$
F(x, y) = \sum^N\sum^M sinc\left( \frac{n}{N} \right)sinc\left( \frac{m}{M} \right)G_{n,m}(x, y)
$$

Dirichlet boundary condition : boundary condition은 우리가 주목할 대상을 위해 경계를 bound하는 것, Dirichlet은 function이 domain의 경계를 따라서 수행해야하는 값을 지정하는 boundary condition. domain의 경계 값을 고정해놓고 쓸 수 있다. 경계의 도함수는 고려하지 않는다.

&ensp;즉 [a, b] x [c, d] 크기에서 $u(x, y)$를 그릴 때, $u(x,c), u(x,d)$나 $u(a,y), u(b, y)$처럼 경계에 있는 값이 0이라면 해당 rectangle의 경계는 값이 모두 0이라는 뜻이다. 본문에서는 우리가 구하는 $F(x, y)$가 Dirichlet boundary conditions(이때 D~ boundary condition과 [D~ condition](https://gosamy.tistory.com/269)도 존재하는데 이것이랑은 다름)을 만족한다는 내용을 시작한다. 이를 통해 cosine 항을 줄일 수 있다. 늘 봐왔듯이 fourier series는 $f(x) = \frac{1}{2}a_{0}+\sum a_{n}c_{n}(x)+b_{n}s_{n}(x)$로 표현할 수 있는데 boundary condition에서 f(0)=f(T)=0이므로 급수 전개식이 $f(x)=\sum b_{n}s_{n}(x)$로 풀 수 있게 된다.

(근데 PSF파트에서 점을 중심으로 even function을 위해 PSF를 cosine항들의 summation으로 가정했었는데 요 파트랑은 어떻게 다른 것인지 아직 잘 모르겠다.)

(other boundary conditions)
Neumann : 요거는 경계 값을 고정해놓지 않고 function의 미분 값이 0으로 세팅
Periodic : 말 그대로 경계 조건이 $u(a+L)=u(x)$처럼 반복되는 것, Dirichlet과 다른 점은 미분 값도 같다는 것.

&ensp;F가 Dirichlet conditions을 만족한다면 F는 각 continuous points들의 fourier series의 sum으로 계산할 수 있다??????????


#### Glare Model

##### 5.1 Square Waves

&ensp;veiling glare는 each ghost가 일정한 강도 $I_{k}$만큼의 2D을 그리며 center line을 통과하는 단면을 boxcar functions의 superposition으로 나타낼 수 있다.

##### 5.1.1 Boxcar Functions

&ensp;ghost가 나타내는 1D intensity function이 \[-r, r] 구간으로 제한할 수 있을 때 boxcar functions의 superposition으로 approximation할 수 있다. Heaviside step function H를 원점 0과 radius가 r이 되도록 대칭시켜 합치면 boxcar function을 구현할 수 있다. 카메라의 움직임에 따라 veiling glare가 따라가기 때문에 temporal interpolation(?)이 효과적인 표현에 중요하다. 그러나 H는 부분적으로 differentiable하지 않기 때문에 뭔가 수정해줘야 한다. 어떤 cosine 함수에 대해 COS(-r) = COS(r) = 0이라면 뭔가 boxcar function의 모양 같기는 하다.

##### 5.2 Global Rotation
