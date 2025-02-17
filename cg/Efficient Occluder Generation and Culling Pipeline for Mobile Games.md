
# method overview

&ensp;end-to-end pipeline을 위한 두 가지 솔루션이 있다. 1. occluder generation과 2. runtime에서 작동하는 occlusion culling, 특히 mobile game에 적합한 모델이다. proxy mesh, proxy structure는 둘 다 복잡한 모델이나 씬, 데이터를 최적화 하기 위한 저해상도 매쉬, 대체 구조 등을 의미한다. Our Mobile-SOC는 이런 occluder(가리는 역할을 하는 개체)와 occludee(가려지는 개체)에게 proxy structure을 create한다. Occluder가 proxy structures로 만들어지고 game assets과 package로 게임 속에 통합된다. runtime에서는 Mobile-SOC가 proxy structre를 통해 효율적으로 occluders를 선택한다. 그리고 depth buffer test를 통해 fully occluded된 mesh를 제외시킨다.

# occluder generation

&ensp;네 가지 주요 requirement가 있다. 1. computational overhead와 memory usage를 줄이기 위해 최소한의 삼각형을 가져야 한다. 2. 높은 수준의 occlusion rate을 가진다. 눈에 보이지 않은 물체를 차단하면서 눈에 보이지만 hidden으로 처리되는 false culling을 최소화 해야한다. "dirty" 한 $M_{I}$는, terrain, mountain 같은 모델의 input mesh, 복잡하고 최적화가 안된 mesh를 뜻한다. 일반적인 dirty mesh는 다음과 같은 점들을 가지고 있다 : intersection, overlap, fracture(쪼개진 여러 조각들), not properly oriented faces(잘못된 면 방향), gaps, open boundary, non-uniform element size(비균일한 요소 크기, 너무 크거나 작음). 이러한 점들에 대해서도 작동해야한다. 4. generation process는 반드시 large scale과 frequent model changes 에서도 효율적으로 수용해야한다. 실제로 개발자들은 absolute occlusion quality보다 generation speed를 더 우선 시 할 수도(might be) 있다. 이를 달성하기 위한 2-step approach. 1. user-defined distance parameter $d$에 따라 offset을 계산한다. 2. offset mesh는 user-specified target triangle count $t$에 맞게 단순화 되어 final occluder $M_{O}$를 생성하게 된다.

## offset mesh generation

&ensp;first step의 목표는 $M_{I}$에서 내부 방향으로 미는 inward offset, 중간 과정의 offset mesh를 만드는 것이다. 이는 false culling을 피하면서 비슷한 occlusion capability를 유지해야한다. typical한 offset의 접근법은 $M_{I}$의 각 vertex $p$의 normal vector 반대 방향으로 $d$만큼 이동하는 것이다. 하지만 dirty input mesh는 normal directions이 unreliable하기 때문에 false culling을 일으킬 수 있다. 따라서 전체 global geometry와 neighboring geometry의 특성을 기반으로 각 $p$에 대한 distinct offset direction $d$를 계산해야한다.

### offset direction

&ensp;$p$의 주변 삼각형(one-ring neighborhood)들의 법선 집합($N(p)$)을 생각하고 offset direction $d$를 구하기 위해 energy function를 maximizing한다. 해당 function에 대해 간략히 소개하자면. 우선 첫 번째 항 $f(n_{i},d)$에 대해서는 $d$가 이웃 삼각형$n_{i}$에 대해 반대 방향을 가지려는 성질을 가지게 한다. 두 번째 항의 $g(M_{I}, p, d)$는 input mesh $M_{I}$와 $p$에서 $d$ 방향으로 ray를 쐈을 때 교점을 구한다. 교점의 법선 벡터가 d와 같은 방향이면 1, 없다면 -1, 다른 방향이면 0을 반환한다. 직관적으로 첫 번째 항은 $p$ 주변의 local feature를 이용했다. $d$가 mesh의 inward 방향을 향하도록 유도했다. 만약 첫 번째 항이 삼각형 개수인, $|N(p)|$보다 작다면 이는 offset 방향으로 인해 일부 이웃 삼각형에 따라 $p$가 outward로 이동했다는 의미이다. 두 번째 항은 $p$를 outward로 이동시키는 방향에 패널티를 주는 $M_{I}$의 global feature을 의미한다. 이는 intersection과 open boundary를 다루거나 offset을 많이 해서 mesh가 뚫리는 경우를 방지한다. inward 하면서 mesh 구조 상 안전한지 test한다.

