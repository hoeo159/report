#### 진행사항

- glare model을 구현하기 위한 2d fourier series의 analytic한 방법 연구

#### 2d fourier series

- 종($\Omega$)모양의 glare model을 표현하기 위해 총 4가지 후보 모델을 산출
	- $z = (1-x^2)(1-y^2)$
	- $z = (\cos x)^2 - (\cos y)^2$
	- $z = 1-(x^2 + y^2)$
	- $z = \frac{1}{2\pi \sigma^2} e^\left( -\frac{x^2 + y^2}{2\sigma^2} \right)$ (2d gaussian)
- 위 두 방법은 구현이 간단한 대신 원형 대칭이 아니고 z = 0 부근의 밑면이 사각형 꼴로 퍼지는 경향이 있음
- 아래 두 방법의 경우, 원형 대칭이긴 하지만 cartesian의 fourier series를 쓸 수 없고 polar coordinate에서 새로운 basis를 쓰는 fourier-bessel series 방법을 써야 한다.

#### $1-(x^2+y^2)$

- coefficient를 구하기 위해 $\int_{0}^{1}r(1-r^2)J_{0}(\alpha_{k}\frac{r}{L})dr$꼴의 경우는 analytic한 적분식으로 표현이 가능
- $1 - (x^2 + y^2)$을 fourier-bessel series로 근사하여 표현
- 아래 그래프는 1d로 잘랐을 때 L = 2.0, N = 5, 10, 20 근사(그래프 범위는 0~10.0)
- ![[F-B series(2, 5).png|200]]![[F-B series(2, 10).png|200]]![[F-B series(2, 20).png|200]]
- 아래 그래프는 1d로 잘랐을 때 L = 5.0, N = 5, 10, 20 근사(그래프 범위는 10.0)
- ![[F-B series(5,5).png|200]]![[F-B series(5,10).png|200]]![[F-B series(5,20).png|200]]
- 아래 그래프는 3d 상에서 봤을 때 비교 시각화(L = 5.0, N = 20 근사)
- ![[3d F-B (5,20).png]]
#### $1-((x-x')^2+(y-y')^2)$

- dx, dy 만큼 이동
- 아래 그래프는 dx = 1.0, dy = -1.0에서 비교 (L = 5.0, N = 20 근사)
- ![[moving F-B series(5, 20).png]]
- y = -1 기준으로 잘랐을 때(각각 N = 5, 10, 20 근사)
- ![[moving F-B series(5,5).png|200]]![[moving F-B series(5,10).png|200]]![[moving F-B series(5,20).png|200]]

#### $A - B((x-x')^2+(y-y')^2)$

- 구간 $L$ 사이에서 glare model의 general form인 $A - Br^2$ 근사
- $J_{0}$의 $\alpha_{k}$의 경우 bessel function에 대한 일반해로 상수로 정해져 있기 때문에 미리 table로 만들어 쓸 수 있다. $J_{0}(x), J_{1}(x), J_{2}(x)$의 경우 
- [bessel function 위키 백과](https://en.wikipedia.org/wiki/Bessel_function)


- ![[bessel_function.png|400]]
- coefficient
$$ \begin{align}
a_{k} &= \frac{2}{L^2 \cdot(J_{1}(\alpha_{k}))^2}\int_{0}^{L}rf(r)J_{0}\left( \frac{\alpha_{k}r}{L} \right)		 \\
&= \frac{2}{L^2 \cdot(J_{1}(\alpha_{k}))^2}\int_{0}^{\sqrt{ A/B }}rf(r)J_{0}\left( \frac{\alpha_{k}r}{L} \right) \\
		&= \frac{2}{L^2 \cdot(J_{1}(\alpha_{k}))^2} \frac{2AL^2}{\alpha_{k}^2}J_{2}\left( \frac{\alpha_{k}}{L}\sqrt{ \frac{A}{B} } \right) \\
&= \frac{4A}{\alpha_{k}^2 \cdot (J_{1}(\alpha_{k}))^2}J_{2}\left( \frac{\alpha_{k}}{L}\sqrt{ \frac{A}{B} } \right)
		\end{align}
		$$
- fourier-bessel series
$$
f(r) \approx \sum_{k=1}^N a_{k}J_{0}\left( \frac{\alpha_{k}r}{L} \right)
$$
- dx = 2.0, dy = -1.0, $z = 9 - 4r^2$, N = 20, L = 10.0
![[gen(10,20,2,-1,9,4).png]]![[gen(10,20,2,-1,9,4)_graph.png]]
-  dx = -1.0, dy = 2.0, $z = 4 - 16r^2$, N = 20, L = 5.0, (아래 graph에서 좌 : N = 20, 우 : N = 40)
![[gen(5,20,-1,2,4,16).png]]
![[gen(5,20,-1,2,4,16)_graph.png|300]]![[gen(5,40,-1,2,4,16)_graph.png|300]]