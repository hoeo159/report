
[해당 글 참조]([valdes.cc - Circular Harmonics](https://valdes.cc/articles/ch.html)) gif로 보면 이해하기가 조금 쉽다
### Spherical Harmonics Intro

 SH는 sphere 상의 surface를 approximation할 때 널리 이용된다. $\theta$와 $\phi$로 이루어진 입력을 받았을 때 sphere surface p에서의 어떤 함수값을 출력할 수 있다. 예를 들어 한 표면 상의 점을 기준으로 가상의 sphere 만든 뒤, 모든 방향에서 들어오는 빛을 approximate 할 때에도 사용된다.

### Circular Harmonics

 CH도 마찬가지로 circular한 틀을 잡고 sphere의 surface처럼 circle 상의 point에 approximating 하는 것이다. 아마 SH와 같은 꼴을 하고 있지 않을까? 또한 azimuth가 없이 specific angle $\theta$를 통해서 Circular Harmonic Series를 구성한다. basis function 역시 $y_{l}^{m}(\theta, \phi)$에서 $B_{i}(\theta)$ 꼴이다.
 $$
 \tilde{F}(\theta) = \sum_{i=0}^{n}w_{i}B_{i}(\theta)
 $$
 hat처럼 ~가 위로 씌어진 것을 tilde function으로 영향력이 작은 차수의 항을 제거한 근사 표현이라는 뜻이다. $B_i$ 역시 $y_{l}^{m}$처럼 level에 따른 basis function을 생성한다. sphere harmonics와 circular harmonics의 basis function이 어떤 하나의 수학적 원리로 설명할 수 있는데 자세한 사항은 추후에 공부해야할 것 같다.
 $$
 B_{i} : \frac{1}{\sqrt{ 2\pi }},\frac{\cos(\theta)}{\sqrt{ \pi }},\frac{\sin(\theta)}{\sqrt{ \pi }},\frac{\cos(2\theta)}{\sqrt{ \pi }},\frac{\sin(2\theta)}{\sqrt{ \pi }},\frac{\cos(3\theta)}{\sqrt{ \pi }}\dots
 $$
fourier에서든 SH에서든 weight를 구하기 위해서 우리가 근사하려는 원본 함수 $F$에 대해 basis와의 곱을 적분해야 한다.
$$
w_{i} = \int _{0}^{2\pi}F(\theta)B_{i}(\theta)d\theta
$$

### Projecting an infinitesimal pulse

너비가 0인 pulse를 표현하기 위해서 Dirac Delta function을 사용했다. fourier에서 pulse를 쓰는 이유는 이해했는데 CH를 분석하는데 $F(\theta)$에 pulse를 쓰는 것에는 어떤 이유가 있을까? $\delta(\theta)$ 함수는 특정 지점($\theta$)를 제외하고는 값이 0이기 때문에 $B_{i}(\theta)$값만 뽑아낼 수 있다.
$$
w_{i} = \int_{0}^{2\pi}\delta(\theta)B_{i}(\theta)d\theta = B_{i}(\theta)
$$
결국 weight를 $B_{i}(\theta)$로 놓고 sum하면 dirac delta function을 apporoximate 할 수 있다는 뜻이다.
![[CH_1.png|700]]
보면 알 수 있듯이 특정 direction에서 강조되는 부분이 있긴 하지만 나머지 부분에서 0이 되어야 하듯 upward와 downward로 뻗어나가는 방향을 볼 수 있다. CH band가 많을 수록 더 정확한 근사가 가능하지만 저자는 그렇게 좋은 근사는 아니었다고 평가했다.

### Projecting a box function

boxcar function은 다음과 같다. pulse와 마찬가지로 무한한 frequency를 가지고 있지만 pulse에 비해 "being smooth"한 부분이 있는 특징이 있다. boxcar function을 위해서 가장 간단하게 생각해볼 때, 일단 기둥이 1인 pulse가 특정 direction 동안 activate되면서 회전하고 다시 step down 되는 생각을 해볼 수 있다. pulse의 wide가 $p_{w}$일 때,
$$
w_{i}=\int_{0}^{2\pi}F(\theta)B_{i}(\theta)d\theta,\quad F(x) = \begin{cases} 1 & \text 0\leq\theta\leq p_{w} \\ 0 & \text otherwise \end{cases}
$$
 펄스 너비를 제외한 구간은 0이기 때문에 다음과 같이 표현할 수 있다. $B_{i}(\theta)$가 단순한 삼각함수이기 때문에 쉽게 표현할 수 있다.($S_{i}(\theta)$)
 $$
 w_{i}=\int_{0}^{p_{w}}B_{i}(\theta)d\theta, \quad w_{i}=S_{i}(\theta)-S_{i}(0)
 $$
 ![[CH_2.png|700]]
 보면 여전히 approximation한 부분이 남아있지만 pulse에 projection 했던 것에 비해 over/undershooting 한 부분이 그렇게 두드러지지 않는 다는 것을 알 수 있다.

### Rotating and Composition CH

 각 Band의 sin, cos 성분을 수직, 수평 성분으로 간주하고 2D vector로 만들어서 회전한다. 이를 통해 band.x, band.y 값을 구하고 new_x, new_y를 표현한다. 각각의 성분이 독립적이기 때문에 add와 subtract역시 단순 합과 차로 계산하여 나타낼 수 있다.
 