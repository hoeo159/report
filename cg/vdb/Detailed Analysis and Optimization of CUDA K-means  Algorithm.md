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
이 연사들은 중간 저장 공간을 절약하기 위해 합쳐질 수(fuse) 있다. 예를 들어 크기가 큰 거리 행렬을 저장하지 않고 바로 최소값 centroid를 찾는 방식 같이... 특정 자료구조 kd-tree 같은게 필터링하는데 도움이 될 수 있지만 병렬처리에는 적합하지 않기 때문에 병렬 처리는 low level에서 구현 최적화에 집중해야 한다. update 파트는 너무 trivial해서 최적화하기 쉽지 않다. 벗!!! centroid set W가 너무 작아서 업데이트 자주해서 write 동기화 파트가 병목이 날 때 느려질 수 있다. 3.3에서 동기화를 피했는데 그렇다할 성능은 못 보였고 원자 연산을 활용해서 동기화를 촘촘하게 하는게 가장 좋은 해결책인 거 같다~

## Related Work

처음에는 glsl 그 다음에 cuda. 메모리와 캐시 사용 패턴을 최적화할 필요성이 많은 연구자들에 의해 제시되었으나, 다양한 구현 전략의 효율성에 대한 종합적인 검토는 아직 부족(so far missing). 