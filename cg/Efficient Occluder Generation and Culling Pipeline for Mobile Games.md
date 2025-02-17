
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

## Occludee Preprocessing

## Runtime Rasterization

## Runtime Visibility Testing

# Experiments