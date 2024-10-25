
## Spherical Harmonic Lighting: The Gritty Details
Robin Green, Sony Computer Entertainment America, Jan 16, 2003

- 논문 리뷰 보다는 번역하면서 정리하는 글이 될 것 같다 사실 논문이 아니다.
- 목적은 우선 Fourier도 좋지만 구면 조화에 대해 더 잘 알고 가면 좋을 것 같아서~이다.

### Introduction

Sphecrical Harmonic(SH)는 3D 모델을 다양한 광원으로 부터 real-time으로 계산하는 테크닉이다. 해당 document을 읽는 사람은 배경지식이 많다고.... 가정한다. 3D 엔진으로 cmmon lighting models을 토딩할 때, CG course에서 배웠듯이 **spectular highlight, diffuse colors, ambient light** 이 세 가지 texture shading을 기본적으로 해야한다. 가장 쉬운 lighting model은 "내적 곱을 이용한 lighting colour"라고 알려진 **diffuse surface reflection model**이다. (여기서는 RGB 값을 표현할 때 intensity라는 표현을 쓴다) 각 광원의 intensity이 단위법선벡터 $\mathbf{n}$과 광원 방향의 단위 벡터 $\mathbf{L}$ 사이를 내적한다.
$$
I = surfacecol*\sum_{i=1}^{n-lights}(lightcol_{i})*(N\cdot L_{i})
$$
위 식을 간단하게 살펴보면 1) $lightcol_{i}$ i번 광원이 들어오는 빛의 양을 구하고 2) $\mathbf{N}$과 $\mathbf{L_{i}}$ 사이의 각도 $\cos \theta$만큼 scaling, 3) surface의 성질에 따른 반사 함수 $surfacecol$에 넣어주는 것이다. 이러한 rendering equation의 문제점은 **real-time**에 할 만큼 **간단한 계산이 아니라는 것**이다. 아래 식은 기본적인 Rendering Equation이다.
$$
L(\mathbf{x}, \vec{\omega}_{0}) = L_{e}(\mathbf{x},\vec{\omega}_{0}) + \int_{\Omega}f_{r}(\mathbf{x}, \vec{\omega}_{i}\to \vec{\omega_{0}})L(\mathbf{x'}, \vec{\omega_{i}})G(\mathbf{x, x'})V(\mathbf{x}, \mathbf{x'})d\omega_{i}
$$
bold 처리되어 있는 것은 vector 성분으로 해당 표면의 위치벡터라고 생각하면 된다. 각 항들에 대해 간략히 설명하면 다음과 같다.

$L(\mathbf{x}, \vec{\omega_{0}})$ : 해당 표면의 한 vertex에서 카메라 방향($\vec{\omega_{0}}$)으로 reflected 된 intensity
$L_{e}(\mathbf{x},\vec{\omega_{0}})$ : object 자체에서 방출하는 빛(light emitted)
$f_{r}(\mathbf{x},\vec{\omega_{i}}\to \vec{\omega_{0}})$ : surface vec x의 BRDF, i번째 광원 방향의 빛에서 camera 방향으로

나머지 $\mathbf{x'}$이 포함된 항은 다른 오브젝트에서 재반사된 빛이 들어오는 것인데 해당 부분은 생략하고 오브젝트 스스로가 방출하는 빛도 일단 없다고 가정하고 다시 써보면
$$
L(\mathbf{x}, {\omega}_{0}) = \int_{\Omega}f_{r}(\mathbf{x},\omega_{i},\omega)L_{i}(\mathbf{x},\omega_{i})(\omega_{i}\cdot \mathbf{n})d\omega_{i}
$$
![[Pasted image 20241017162044.png|300]] [위키피디아](https://en.wikipedia.org/wiki/Rendering_equation)참조

갑자기 flux에 대한 개념이 등장한다. 단위 시간 dt 동안 어떤 면적 A를 투과하는 광자의 개수를 flux라고 한다. 다시 말해서 단위 시간동안 면적에 빛이 해준 양이기 때문에 단위는 J/s, 일률 watts.  중요한 부분은 dt동안 광자가 A만큼 면적을 통과하기 때문에 부피의 개념이 들어가야한다. 아래 두 번째 사진처럼 진행 방향과 dA와의 각도가 얕을 수록  쓸어가는 부피가 작아진다, 바닥의 projection area를 참고해보자($\cos \theta$ scale). 이때 단위면적 dA를 생각해서 **irradiance**를 구할 수 있다.

![[Pasted image 20241017164045.png|300]]   ![[Pasted image 20241017164102.png|172]]    $E = \frac{d\Phi}{dA}$

이제 어떻게 이 한 점 x를 반구형으로 영역을 놓았을 때 들어오는 빛들을 감지하고 integral하여 real-time에 계산할 수 있을까??가 이 문서의 목표가 되겠다. 이 저자는 특히 게임 프로그래머에 대한 말이 많다.

### Monte Carlo Integration

셰이더에서 적분 계산을 어떻게 하는가?를 위해서 Monte Carlo Integration가 등장한다. Monte Carlo Method에서 가장 유명한 것은 랜덤 값을 이용해서 원을 그리는 것이 있다. 주사위를 던질 때에도 많이 던질 수록 평균값은 기댓값에 수렴하게 된다. 특정 식 $f(x)$를 적분하고 싶을 때 x가 특정 값으로 선정될 확률을 p(x)라고 한다면 아래와 같은 성질이 성립한다.
$$
\int_{-\infty}^{\infty}p(x)dx = 1, \quad \int_{a}^{b}f(x)dx = \int_{a}^{b} {\frac{f(x)}{p(x)}}\cdot p(x)dx
$$
rhs를 잘 보면 $f(x)/p(x)$의 기댓값을 구하는 식임을 알 수 있다. 따라서 풀기 까다로은 적분식을 여러 번의 시행을 통해 값을 유추할 수 있다는 뜻이다.
$$
\int f(x)dx = \int{\frac{f(x)}{p(x)}p(x)dx}  \approx E\left[ \frac{f(x)}{p(x)} \right] = \frac{1}{N}\sum_{i = 1}^N \frac{f(x)}{p(x)}
$$
따라서 f(x)의 샘플 x를 많이 뽑으면 적분 값을 추측할 수 있다는 뜻이다.