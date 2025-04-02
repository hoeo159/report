# Abstract

적당한 k-means 소개, 3가지 steps : nearest neighbor finding, efficient distance computation, cluster-wise reduction.  그 다음 in this work ~ 많은 연구들이 이미 있지만 gpu 구현의 중요한 성능 측면에 아직 해결되지 않은 점을 찾았다. cluster 수 K, dataset 크기와 차원 -> memory bandwidth, 캐시 제한과 workload dispatching의 최적화. 개별 step에 대한 상세한 분석을 제시하고 몇 가지 최적화를 제안한다~

# Introduction

K-means는 voronoi cell 파티셔닝 np hard 문제. Euclidean metric으로 이루어진 벡터 공간, 거리 등 어려운 용어를 쓰는데 결국 centroid 제곱 오차를 최소하는 게 목표라는 뜻. 여러 응용 알고리즘들이 있었다 : k-median, k-medoids, self-organizing maps, SOM 등. k-means를 실생활에 많이 쓴다, 그 자체 알고리즘 뿐 아니라 개별 step에 적용되는 아이디어 들도~

## Employing GPUs

k-means는 단순하기 때문에 병렬 처리에 적합하다. 메인 기본 points는 다음과 같이 요약

- Core occupancy

연산 성능을 영끌하기 위해서 충분한 스레드를 생성해서 모든 코어가 일하도록 해야한다. 보통 oversubscription이라고 코어보다 많은 스레드를 생성하면 코어가 일 안하고 멈출 때 빠르게 스레드를 전환하는 방법을 통해 코어 점유율을 높인다.

 - Communication locality

코어, SM, block, shared memory에 대한 용어 설명 한 SM에 할당할 수 있는 스레드 수는 레지스터 크기와 같은 여러 요인에 의해 제한될 수 있다.

- Warp divergence

warp에 대한 용어 설명, if 같은 branch가 있으면 일부 thread 명령이 무시되어 성능 저하를 초래할 수 있다.

- Memory efficiency

공유 메모리나 이런거 잘 써야 한다. memory bandwidth는 넓지만 core를 동시에 지원하기 때문에 문제가 생길 수 있다.

## Outline and Contributions

1. detailed analysis와 microbenchmark 제공 각 step 마다. 다른 병렬화 문제에도 쓸 수 있을 것이라 기대
2. 이 분석 결과를 활용해서 현재 알고리즘들보다 성능이 뛰어난 k-means 구현
3. 오픈 소스로 공개해서 추가적인 연구

이런게 contribution이 될 수 잇나?
그 뒤에는 outline 소개, section 2 : k-means 알고리즘 관련 연구 검토 3 : 개별 setp 성능 분석과 구현을 위한 마이크로 벤치마크 결과 4 : 최적화 구현과 성능 평가 결과 5 : 결론

# K-means algorithm

k-means 알고리즘에 대한 간략한 설명 1. assignment step 2. update step 근데 행렬 용어를 써서 더 어렵게 설명하고 있다

## Implementation Strategies

연산 순서
- 데이터 X와 centroid W 사이의 유클리드 거리 계산(거리 행렬이라고 표현)
- X 각각의 거리를 기반으로 가장 가까운 W를 찾아 벡터 N을 구상한다
- update에서는 cluster 마다 데이터 합 계산해서 새로운 centroid를 만든다

이 연산들은 중간 저장 공간을 절약하기 위해 합쳐질 수(fuse) 있다. 예를 들어 크기가 큰 거리 행렬을 저장하지 않고 바로 최소값 centroid를 찾는 방식 같이... 특정 자료구조 kd-tree 같은게 필터링하는데 도움이 될 수 있지만 병렬처리에는 적합하지 않기 때문에 병렬 처리는 low level에서 구현 최적화에 집중해야 한다. update 파트는 너무 trivial해서 최적화하기 쉽지 않다. 벗!!! centroid set W가 너무 작아서 업데이트 자주해서 write 동기화 파트가 병목이 날 때 느려질 수 있다. 3.3에서 동기화를 피했는데 그렇다할 성능은 못 보였고 원자 연산을 활용해서 동기화를 촘촘하게 하는게 가장 좋은 해결책인 거 같다~

## Related Work

처음에는 glsl 그 다음에 cuda. 메모리와 캐시 사용 패턴을 최적화할 필요성이 많은 연구자들에 의해 제시되었으나, 다양한 구현 전략의 효율성에 대한 종합적인 검토는 아직 부족(so far missing).
Lutz et al. 에서는 single-pass로 짜는게 중요하다 cpu의 50배, double-pass의 2배 성능을 보였다. 근데 "fused kernels"를 이용한 single-pass process는 small k-means instance를 한 번에 계산하는 데 유용하다. Cuomo et al. 에서는 cpu와 gpu 메모리 전송 성능 비용을 평가하지만 shared memory 같은 계산 관련 최적화 효과는 무시한다(의미 없다~).
YinYang k-means 알고리즘의 gpu 버전이 가장 SOTA. YY는 Lloyd algorithm이라고 불리는 naive한 k-means를 최적화 했지만 gpu 에서는 overhead 때문에 최적이 아니다. Nelson and Palmieri에서는 global과 shared memory 사이 tradeoffs를 집중하고 memory locking을 이용한 thread 동기화 모델을 제시. YY의 kmcuda 보다 20배~40배 성능 향상, 하지만 increased dimensionality of data에 대해 overflow of shared memory에 대해서는 다루지 않았다.

# Performance Analysis

best 구현을 위해서 각 operations과 각 bottleneck에 대한 solution을 제시해야한다. 우리의 code는 general 하지만 parameter을 reasonable하게 하기 위해서 가정이 몇 개 있다.
- data dimension $d$를 max 128로 설정했다. 고차원 공간에서 Euclidean distance는 효율성이 떨어지고 보통 data clustering 이전에 pca나 random projection 처럼 차원 축소를 보편적으로 쓰기 때문이다.
- cluster 수 k는 16~4096, 논문에서 다뤄지는 대부분의 예시를 커버한다.
- total dataset이 gpu 메모리에 충분히 저장될 수 있다고 가정. 데이터를 분할하고 gpu에 롤리는 기법이 있지만 논문에서 이 부분을 다루지 않았다. 모든 코어를 쓸 수 있을 만한 적당히 큰 n을 사용했다.

assignment step, update step 두 단계를 개별적으로 최적화 하는데 중점을 두었다. 모두 thread 수와 workload assignment 방식, data layout, caching 등 메모리 최적화 전략에 대해 논의한다. 기본적인 memory layout에는 두 가지가 있다. 
Array of Structure : 각 데이터 vector를 연속된 메모리 블럭에 저장. 
Structure of Array : 벡터의 각 차우너을 독립적인 연속된 메모리 블럭으로 저장. 같은 차원의 데이터끼리는 인접하게 저장.

update step의 efficiency는 write-synchronization이 중요하다. update가 전체 실행 단계에서 차지하는 비율은 작지만 k가 적을 때에는 병목이 될 수도 있다. atomic operations을 이용해서 fine-grained 동기화를 촘촘하게 넣는게 효율적이다~

## Hardware and Methology

CPU 성능 고려 x, GPU 커널 실행시간만 측정함, 5회 반복 평균값. 절댓값이 아니라 상대적 비교에 사용되었다. gpu는 Maxwell~Volta 세대만 썼지만 공통된 패턴이 있어서 일반성이 확보됨. 매우 저사양에서는 결과가 다르게 나올 수 있음을 암시 laptop이나 portable device. 성능 비교를 위해 연산량에 따라 정규화 했다. 예를 들어
- assignment step = O(nkd) -> 정규화하면 T / nkd
- update도 T / nd
- 전체 알고리즘 iteration이 I일때 T / nkdI
차원 수, 클러스터 수, 데이터 수가 다른 다양한 실험 조건에서 공정한 비교.
n은 수 백만 개 수준. CUDA block 당 스레드 수는 경험적으로 선택했다. reg 사용이 많은 알고리즘은 256 threads, 그 외에는 1024 threads.

## Assignment Step

데이터 포인트를 가까운 centroid에 assign. 이 연산은 독립적이므로 수행할 수 있다. naive하게 하면 cache 적중률이 낮아 memory bandwidth와 latency에 성능이 제한된다. 우선 naive하게 baseline을 해놓고 최적화를 하겠다~. Assignment step은 multi-query search와 비숫하고 pivot-based approximate NN index construction과 비슷하다. query와 pivot에서 유클리드 거리를 쓴다. assn step은 거리 계산하는 데이터 접근 패턴에서 행렬곱처럼 모든 데이터를 순회하는 유사한 데이터 접근 패턴을 가지지만 가장 가까운 cetroid indexing을 하는 과정에서 캐싱 전략이 유효하다.
일반적으로 캐싱 전략이라 함은 캐시 미스로 인해 발생하는 데이터 로딩과 작업 간 조절인데~ gpu 캐시는 상대적으로 작고 동작 방식이 특이해서 좀 어렵다. 이러한 복잡성을 완화하기 위해 shared memory에 수동으로 caching하고 더 많은 데이터를 register에 넣는다. shared memory는 좋은데 크기가 제한적이다. SM은 각자 레지스터 풀을 가지고 있고 thread가 이걸 써서 저장, 연산한다. 이것도 크기는 제한되어 있고 너무 많은 thread가 동시에 reg를 쓰려고 하면 병목이 발생한다. 따라서 좋은 메모리는 크기가 제한적이기 때문에 다양한 캐싱 전략과 메모리 접근 패턴 최적화를 탐구해야한다. 특히 communication locality를 높이는 것이 중요하다.

### overview of optimizations

두 가지 objective가 있다. multiple core 문제에서 최적으로 연산 작업들을 나눈 것과 이러한 분배 작업에 적합한 캐싱 전략을 디자인 하는것이다. assn step에서 세가지 축으로 나누어 설명.
- N point에 대한 병렬화 -> baseline 단계에서도 구현 가능 각 점은 독립적 계산이기 때문
- k개 mean에 대한 병렬화, centroid 수 만큼 병렬적으로 연산 수행
- d 차원에 대한 병렬화, 각 차원 당 n k 번의 벡터 연산을 수행한 후 각 point에 최소를 찾기
O(nkd)이기 때문에 이런 3가지 방법을 생각할 수 있다. 이후 n과 k에 대한 병렬화 조합에 초점을 맞추고 있다. 캐시 활용도 향상에 도움이 되기 때문??? d에 대한 병렬화는 각 차원에 대해 모든 데이터 포인트를 조사하기 때문에 캐시 히트가 낮아질 수[있다](본문에서 인용한 Employing GPU architectures for permutation-based indexing 연구에 따르면, L2 거리 계산 시 d에 대한 병렬화는 오버헤드가 높습니다. 이는 차원이 증가할수록 캐시 활용도가 떨어지고, 메모리 접근 비용이 증가하기 때문일 수 있습니다.) .