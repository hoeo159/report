
# method overview

&ensp;end-to-end pipeline을 위한 두 가지 솔루션이 있다. 1. occluder generation과 2. runtime에서 작동하는 occlusion culling, 특히 mobile game에 적합한 모델이다. proxy mesh, proxy structure는 둘 다 복잡한 모델이나 씬, 데이터를 최적화 하기 위한 저해상도 매쉬, 대체 구조 등을 의미한다. Our Mobile-SOC는 이런 occluder(가리는 역할을 하는 개체)와 occludee(가려지는 개체)에게 proxy structure을 create한다. Occluder가 proxy structures로 만들어지고 game assets과 package로 게임 속에 통합된다. runtime에서는 Mobile-SOC가 proxy structre를 통해 효율적으로 occluders를 선택한다. 그리고 depth buffer test를 통해 fully occluded된 mesh를 제외시킨다.

# occluder generation

&ensp;네 가지 주요 requirement가 있다. 1. computational overhead와 memory usage를 줄이기 위해 최소한의 삼각형을 가져야 한다. 2. 높은 수준의 occlusion rate을 가진다. 눈에 보이지 않은 물체를 차단하면서 눈에 보이지만 hidden으로 처리되는 false culling을 최소화 해야한다. "dirty" 한 $M_{I}$는, terrain, mountain 같은 모델의 input mesh, 복잡하고 최적화가 안된 mesh를 뜻한다. 일반적인 dirty mesh는 다음과 같은 점들을 가지고 있다 : intersection, overlap, fracture(쪼개진 여러 조각들), not properly oriented faces(잘못된 면 방향), gaps, open boundary, non-uniform element size(비균일한 요소 크기, 너무 크거나 작음). 이러한 점들에 대해서도 작동해야한다. 4. generation process는 반드시 large scale과 frequent model changes 에서도 효율적으로 수용해야한다. 실제로 개발자들은 absolute occlusion quality보다 generation speed를 더 우선 시 할 수도(might be) 있다. 이를 달성하기 위한 2-step approach. 1. user-defined distance parameter $d$에 따라 offset을 계산한다. 2. offset mesh는 user-specified target triangle count $t$에 맞게 단순화 되어 final occluder $M_{O}$를 생성하게 된다.

## offset mesh generation

&ensp;first step의 목표는 $M_{I}$에서 내부 방향으로 미는 inward offset, 중간 과정의 offset mesh를 만드는 것이다. 이는 false culling을 피하면서 비슷한 occlusion capability를 유지해야한다. typical한 offset의 접근법은 $M_{I}$의 각 vertex $p$의 normal vector 반대 방향으로 $d$만큼 이동하는 것이다. 하지만 dirty input mesh는 normal directions이 unreliable하기 때문에 false culling을 일으킬 수 있다. 따라서 전체 global geometry와 neighboring geometry의 특성을 기반으로 각 $p$에 대한 distinct offset direction $d$를 계산해야한다.

### offset direction

&ensp;$p$의 주변 삼각형(one-ring neighborhood)들의 법선 집합($N(p)$)을 생각하고 offset direction $d$를 구하기 위해 energy function를 maximizing한다. 해당 function에 대해 간략히 소개하자면. 우선 첫 번째 항 $f(n_{i},d)$에 대해서는 $d$가 이웃 삼각형$n_{i}$에 대해 반대 방향을 가지려는 성질을 가지게 한다. 두 번째 항의 $g(M_{I}, p, d)$는 input mesh $M_{I}$와 $p$에서 $d$ 방향으로 ray를 쐈을 때 교점을 구한다. 교점의 법선 벡터가 d와 같은 방향이면 1, 없다면 -1, 다른 방향이면 0을 반환한다. 직관적으로 첫 번째 항은 $p$ 주변의 local feature를 이용했다. $d$가 mesh의 inward 방향을 향하도록 유도했다. 만약 첫 번째 항이 삼각형 개수인, $|N(p)|$보다 작다면 이는 offset 방향으로 인해 일부 이웃 삼각형에 따라 $p$가 outward로 이동했다는 의미이다. 두 번째 항은 $p$를 outward로 이동시키는 방향에 패널티를 주는 $M_{I}$의 global feature을 의미한다. 이는 intersection과 open boundary를 다루거나 offset을 많이 해서 mesh가 뚫리는 경우를 방지한다. inward 하면서 mesh 구조 상 안전한지 test한다. $d$의 후보군은 unit sphere 상에서 찾는다. 샘플링해서 여러 개의 벡터 방향을 생성하고 각 방향에 대해 energy function 값을 계산, max $d_{i}$를 선택한다.

### offset distance

&ensp;이전에 측정했던 $\sum_{i=1}^{|N(p)|}f$를 이용해서 boundary facing a large open space 또는 surrounded by a neighborhood of improperly oriented triangles, 즉 부적절하게 정렬된 삼각형으로 둘러싸인 경우에는 offset distance를 0으로 설정해 이동하지 않는다. 또한 g 값이 -1 즉, $p$에서 $d$방향으로 ray를 쐈을 때 mesh와 교차하지 않은 경우도 open space기 때문에 이동하지 않는다. 그 외의 경우에는 mesh와 처음 교차하는 거리 $e$를 이용해서 $min\left( d, \frac{e}{2} \right)$, mesh의 두께를 고려한 사이즈로 이동한다.

## simplification

&ensp;offset mesh $M_{s}$를 만들었다면 conservative simplification을 적용해서 final occluder mesh $M_{o}$를 만들어야한다. standard mesh simplification pipeline -> 반복적인 edge collapse를 수행해서 삼각형 수를 목표 값 $t$로 줄이는 과정 + 2가지 새로운 contribution을 추가했다. 사유는 highly efficient low-poly occluder를 생성하면서 false culling을 방지하기 위해서다.

&ensp;첫 번째는 thin and small regions triangles을 제거하는 것이다. 얇은 삼각형을 mesh의 나머지 부분과 분리해서 neighbor triangle에서도 분리한다. "thin"하다는 것의 평가 지표에는 두 조건을 만족해야 한다. 1) 삼각형 세 변 중 가장 긴 변의 길이가 $\frac{2A}{d}$ 2 * 면적/offset 거리 보다 길고 2)  삼각형 중 at least 정점이 $\frac{e}{2} < d$ 인 경우, mesh가 너무 얇아서 self-intersection 할 가능성이 있기 때문이다. 이런 두 조건을 만족하는 삼각형을 줄임으로써 occlusion 성능을 유지하면서 연산 비용과 false culling을 줄일 수 있다.

&ensp;두 번째는 simplified mesh가 $M_{I}$(input 원본 dirty mesh)의 내부에 있는 상태(interior)를 유지하도록 강제하는 것이다. QEM(Quadric Error Metric)기반 edge collapse는 mesh simplification에 많이 쓰지만 보장하지 않는다. 어떤 걸?? simplified mesh가 기존 mesh의 bound 전체를 남기질 않기 때문에 false culling를 초래한다. variants of QEM은 conservative하게 제약을 추가하거나 충돌감지와 결합해서 해결하지만 input mesh에 defects가 포함되어 있을 수도 있다. 이러한 defects는 offset mesh인 $M_{s}$와 $M_{I}$가 intersects 하는 경우 문제가 생길 수 있다. 추가적으로 이러한 접근은 계산 overhead가 크다. 이러한 기존 QEM은 vertex가 원래 mesh의 plane과 최대한 가까운 위치로 유지되도록 하는 방식이다.

&ensp;대신에 QEM에서 최소한의 변화만 준다.(띄어쓰기 안됨) 핵심 아이디어는 simplification 과정을 편향시킨다. supporting planes를 backward로 이동시킴으로써. 이는 vertex들을 conservatively하게 position되는데 간접적으로 기여한다. 기존의 평면의 방정식
$$
q = (x, y, z, w)
$$
에서 평면의 방정식을 보수적으로 수정해서 w 항에 bias 값을 추가한다.
$$
q = (x,y,z,w+\epsilon)
$$
기존에서는 한 점 $p = (x, y, z, 1)$에 대해 해당 점과 평면 사이의 거리의 제곱이 오차함수로 정의된다. 정점 p가 포함된 모든 평면에 대한 거리 오차를 합산하는 방식이다.이를 최소화 하도록 정점을 병합한다.
$$
E(p)=p^T Q p, Q=\sum_{i}q_{i}q_{i}^T
$$
수정된 새로운 오차 함수는 다음과 같이 표현된다.
$$
\bar{E}(p)=\bar{p}^T(q_{\epsilon}q_{\epsilon}^{T})\bar{p}
$$
평면이 좀 더 바깥쪽으로 밀어져서 더 뒤에 있는 정점을 선택하게 되어 mesh 내부로 수축하는 것을 방지 할 수 있다. 또한 QEM은 정점 병합을 수행 할 때 가까운 정점들을 하나로 통합하는데 Occluder로 사용될 영역이 손실될 수 있다. bias를 추가하면 occlusion 영역을 보호하면서 삼각형 개수를 줄일 수 있다고 한다. 또한 거리 오차를 최소하하는 방향으로 이동할 때 좀 더 보수적인 방향으로 이동하도록 돕는다. epsilon의 값은 bounding box volume의 세제곱근에 비례한다.

# Mobile-SOC : Occlusion Culling

&ensp;모바일을 위한 software occlusion culling 기법을 처리한다. 현 솔루션 Intel Masked SOC와 UE4 SOC는 모바일 플랫폼에 유망하지만 아직 낮은 성능을 보이고 있다. 몇가지 technical contributions들로 이루어져 있다 : Occluder Preprocessing, Occludee Preprocessing, Mobile-Specific Rasterization Optimization, Advanced Visibility Testing. 많은 작은 mesh들이 다른 mesh들을 막을 수 없기 때문에 occluder이 없고, 몇 mesh는 occluder이면서 occludee일 수도 있다.

## Occluder Preprocessing

&ensp;기존의 SOC 알고리즘은 occluder 삼각형을 cluster로 group한다. 처음에는 두 삼각형을 quad로 병합하고 SAH 기반 bound box hierarchy 구조를 이용해서 quad와 삼각형을 bounded size 크기의 클러스터로 group화 한다. 여기서 다른 타입의 cone cluster를 도입한다. 이는 cluster의 모든 삼각형이 back-facing을 향하고 있을 때 해당 cluster를 스킵하는 기술에서 영감을 받았다. mesh rendering에서 일반적으로 사용된다고 한다. such cluster를 만드는 방법 : 이웃 삼각형 수 기준으로 삼각형들을 sorting. 가장 많은 neighbor를 가진 삼각형부터 병합을 시작. progressively하게 인접한 삼각형과 결합해서 cone cluster를 만든다. 추가되는 삼각형의 normal direction이 cone's axis와 가까워질 때까지 진행한다. 만약 cone's spanning angle이 predefined된 threshold에 도달 하면 중지. 이 콘들은 visibility test에서도 간단한 벡터 연산만 하기 때문에 모바일에서도 런타임이 간단하다. cone
s angle이 너무 작으면 back-facing이고 culled 될 가능성이 작기 때문에 기존의 standard cluster로 처리해야한다.

## Occludee Preprocessing

&ensp;Visibility Test <- occludee의 AABB 8 corner 점을 clip space에 projecting 해서, 가장 가까운 depth value를 결정하고 이 값을 depth buffer에 비교해서 가려졌는지 확인하는 방식. 하지만 너무 conservative해서 AABB가 true geometry에 비해 poorly 근사했을 때 불필요한 랜더링이 수행된다. 따라서 multi-AABB test를 제안. occludee preprocessing 동안, SAH 알고리즘을 적용해서 occludee의 traingles를 더 작은 AABB로 cluster한다. 이런 AABB들을 $S_{n}$으로 n개의 AABB set으로 관리. $S_{n}$의 quality을 test 수행. $S_{n}$을 여러 번 생성해서 최적의 $S_{n}$ AABB 집합을 찾는다. Occludee의 모든 AABB가 가려졌을 때 최종적으로 culling 판단. Occludee의 형상이 단순하면 기본 AABB를 사용하고, 복잡하면 Multi-AABB를 적용한다.

## Runtime Rasterization

Rasterization 과정이 AVX256 기반 명령어 집합에 의존해서 모바일 환경에 incompatible했다. cross-platform XSIMD library를 이용했고 128 bit NEON inst를 이용해 최적화를 진행했다. 추가적으로 cluster assembly 과정에서 vertex position를 caching하고 pre-fetching하는 것을 통합해서 중복 계산을 최소화하고 low-end mobile 성능을 향상시켰다.

## Runtime Visibility Testing

256 x 144 사이즈의 낮은 해상도 depth buffer를 사용했다. 대부분 occludees에는 잘 작동하지만 decal 같은 작은 물체는 카메라가 움직을 때 flickering 같은 결함이 발생한다. standard solution은 AABB에 scale factor를 추가해서 확장하거나 depth test하는 동안 투영되는 영역을 높이는 방법을 쓴다. 벗 이러한 접근은 종종 culling 정확도를 낮춘다, 투영되는 pixel 면적을 증가시키기 때문이다. Mobile-SOC에서는 visibility score라는 기법을 도입했다. 각 occludee에 대해 floating-point valued 인 점수 $v$를 유지한다. 처음에는 $v = 1$로 객체가 보이는 상태를 의미한다. depth test에서 만약 객체가 가려지지 않았다면 $v = 1$로 즉시 보이게 만들 수 있다. occludee가 한 프레임에서 가려지는 경우 $v$를 절반으로 감소해서 천천히 사라지도록 smoothly blend out 하도록 만든다. $v\leq{\frac{1}{16}}$이 될 경우 completely하게 culling을 수행한다. 이를 통해 작은 객체의 flikering 현상을 방지하고 부드러운 전환으로 false culling을 방지한다. 또한 낮은 resolution의 depth buffer로도 효과적인 occlusion culling 수행을 보여준다.

# Experiments

effectiveness, efficiency, runtime performance를 측정한다. 측정 대상은 occluder generation과 occlusion culling system이다. High, Mid, Low-end 수준의 mobile device에서 측정했다. offset distance는 $d = 0.5\%l$, $l$은 $M_{I}$ input mesh의 bounding box 대각선 길이를 의미한다.

## Occluder Generation

Valid Rate : occluder가 원본 mesh와 동일한 occlusion 효과를 가지는 비율
Error Rate : 반대로 occlusion 잘못 수행하는 비율
dataset : 85개 모델, 건축 구조, 게임 scene, terrain 등을 사용했다. occluder가 필요없는 low-density한 mesh는 제외했다. 평균적으로 87413개의 face(삼각형)과 일부 모델은 50만 개 이상의 삼각형을 포함한다. occluder generation이 다양한 모델에서 일관된 성능을 제공하는지 확인. $M_{I}$의 삼각형 개수에 따라 occluder 자동으로 조절. $t=max(300, 1\%|M_{I}|)$. larger size, complex structure에서는 더 많은 삼각형 할당. 평균 실행 시간 : 23.8s(가장 느린 모델은 178s), Valid Rate : 70.2% ~ 98.3%, 평균 82%, Error Rate (오류율): 0.001% ~ 2.0%, 평균 0.27%

기존 연구와의 비교 (Silvennoinen et al. 2014)
Silvennoinen et al. (2014) 방법은 전체 85개 모델 중 단 15개만 성공적으로 Occluder를 생성. 실행시간도 328초에 Valid Rate 68.1%, Error Rate가 21.3%, false culling이 많이 발생. 부족했던 점은 outside에서 accessible한 inner structures를 식별하는 능력이 부족했다. Mobile-SOC 방식에서는 동일 15개 모델에서 Error Rate 0.0019%로 감소시켰다.

Wu et al. 2022
Wu et al. 2022의 occluder 생성 과정 설명..
planar face를 기반으로 candidate triangles를 선택하기 때문에 개수가 insufficient 하다. Iso-surface는 불필요한 occlusion을 발생시켜 false culling을 일으킨다. iterative triangle removal 단계의 연산 속도가 느림. 몇 시간 이상 소요될 수 있다. 비교 정확성을 위해 collection step만 test했다. second step은 valid rate를 낮추기 때문에 우리의 바법이 더 좋았다. 공정성을 위해 occluder의 삼각형 수를 갖게 맞추었다. error rate는 3.32% -> 0.32%, 실행시간은 Wu et al. 2022가 10초 정도 더 빠르지만 정확도는 Mobile-SOC가 우수함.  Wu et al. 2022는 85개 dataset 중 1개가 무한 루프에 걸렸기 때문에 Mobile-SOC는 84개 dataset과 비교했다.

Chen et al. 2023
Chen et al. 2023는 low-poly mesh를 생성하는 robust method를 소개함. occluder로 활용할 가능성이 있음. 하지만 삼각형 개수를 조절할 수 없다는 단점이 있음. 비교를 위해 Chen et al. 2023 방식을 실행해서 occluder를 생성하고 동일한 삼각형 개수를 가지도록 Mobile-SOC 방식을 적용하여 비교. Chen et al. 2023 방식이 valid rate가 87.6%로 높았지만 error rate가 7.5%로 높았다. Mobile-SOC를 넣었을 때에는 error rate가 0.47%로 낮출 수 있었다. 속도 역시 101.8초에서 25.2초로 단축시켰다. Chen et al. 2023 방법을 개선한 modified version으로 test를 추가로 진행. inner iso-surface를 계산한 후 QEM 기반으로 삼각형을 단순화. 하지만 self-intersection이 발생하고 thin wall이 손실되어 작은 fragment로 분할되어 나타났다. 실행 결과는 valid rate가 42.8%로 크게 낮아졌다. error rate는 3.24%로 여전히 높다고 판단. 수정된 버전에서도 Mobile-SOC가 더 좋은 성능을 가졌다.

parameter study
$d$(offset distance), occluder가 input mesh에서 얼마나 안쪽으로 들어가는지 조절하는 요소. d가 크면 false culling이 감소, d가 작다면 valid rate가 증가하는 면이 있다. 따라서 $d$ 값을 조절해서 최적의 occluder를 생성할 수 있는지 평가. offset distance d를 다르게 설정하여 여러 Occluder를 생성하고, Valid Rate 및 Error Rate를 비교 분석. 0.5%, 1.0% 5.0% size. 모델의 특성에 따라 $d$값을 조절 하면 최적의 occluder를 생성할 수 있다.
## Mobile-SOC

평가 지표 : cpu time, draw call number(occlusion culling 이후 rendering해야하는 객체 수), FPS
비교군 : 기존 occlusion culling 기법, UE4 SOC, HOQ
테스트 방식 : unreal engine 5.4 demo project 사용, sun temple 오픈 소스 레벨에서 성능 평가, 높은 밀도의 3d 모델을 복사하여 복잡한 게임 씬 생성 후 테스트.

결과 : Mobile-SOC가 UE4 SOC보다 최대 4.6배 빠르고 draw call 수가 적어 gpu 부하가 감소. fps 성능도 향상된 모습을 보였다. HOQ는 gpu에서 실행하기 때문에 cpu time이 없고 1 frame delay이 발생하여 visibility test 오류가 발생할 수 있다. Mobile-SOC는 실시간으로 판단하기 때문에 정확한 culling이 가능하다. 즉 HOQ가 성능이 더 좋거나 비슷할 수 있지만 정확도가 떨어진다.

데모에서도 노란색 픽셀이 많을 수록 더 많은 occluder mesh의 face가 outside로 나왔음을 의미하고 더 높은 error rate를 나타낸다. 우리의 방법과 비교했을 때,  Silvennoinen et al. 2014 (figure 18(e))는 error rate가 가장 높은 반면, the old version of our method는 가장 낮다. 하지만 몇 모델은 낮은 valid rate를 생성할 수 있다. figure 18g에서 확인할 수 있다.

# Conclusion

이 논문은 mobile game 최적화 rendering solugion을 제공한다. 자동 occluder generation과 효율적인 runtime occlusion culling 개념이 도입됬다. 다양한 dataset과 mobile platform에서 robust한 performance를 보여주었다. 우리 solution이 여러 사내 모바일 게임에 이미 배포되어 있고 수만 개의 복잡한 모델을 처리하고 있다. 하지만 몇가지 limitation이 존재. 많은 identified tiny triangles로 구성된 객체를 처리할 때 한계가 있고 이는 valid rate 감소로 이어진다. figure 16에서 확인할 수 있다. 이러한 문제를 해결 하기 위해 thin triangle 감지 단계를 비활성화 할 수 있다. 우리는 이런 이슈들을 다루는 더 자동화된 솔루션을 연구 중이다. 
