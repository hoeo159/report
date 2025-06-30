### 논문 진행 사항
#### cluster 중심 수렴 실험 결과
+ 전체 cluster 수 = 25, Data 수 = 100000, 1536 차원, 10번씩 진행
+ 기본 kmeans - Average results over 10 runs
	+ Global SSE: 95,655.79
	+ Total execution time: 207,280.17 ms
	+ Convergence: iteration 151.2
		+ 8번째 실험 결과
		  ![[result-diff-per-iter-standard.png|400]]
+ Hierarchical kmeans - Average results over 10 runs
	+ Global SSE: 95,685.27
	+ Total execution time: 116,924.64 ms
	+ Coarse stage convergence: iteration 85.6 
		+ 6번째 실험 - coarse clustering 결과
		  ![[result-diff-per-iter-hierarchical-coarse.png|400]]
	+ Fine stage convergence: iteration 72.56 
		+ 6번째 실험 - fine clustering 결과
		  ![[result-diff-per-iter-hierarchical-fine.png|400]]
#### 추가적인 성능 지표 구현
+ Calinski–Harabasz Index
	+ label 없이 학습된 clustering이 얼마나 잘 clustering 되었는가를 평가하는 데 사용됨
	+ 좋은 clustering이라고 평가하는 기준
		+ 각 cluster 안에 데이터가 서로 가깝게 모여 있어야 함
		+ 서로 다른 cluster는 충분히 떨어져 있어야 함
	+ Calinski–Harabasz Index는 위 두 가지를 수학적으로 동시에 고려하는 지표
	+ 정의 (https://en.wikipedia.org/wiki/Calinski%E2%80%93Harabasz_index)
	  ![[Calinski–Harabasz-index-definition.png]]
		+ BCSS: Between-Cluster Sum of Squares
		  ![[bcss-definition.png]]
			+ n_i: the number of points in cluster C_i
			+ c_i: the centroid of C_i
			+ c: the overall centroid of the data
			+ 각 cluster 중심이 전체 데이터의 평균에서 얼마나 떨어져 있는지를 나타냄
			+ cluster들이 전체 공간에 얼마나 잘 분리되어 있는지를 보여줌
			+ cluster 중심들이 서로 멀리 떨어져 있을수록 값이 큼
		+ WCSS: Within-Cluster Sum of Squares
		  ![[wcss-definition.png]]
			+ 각 cluster 안에서, 포인트와 해당 cluster 중심 간 거리의 제곱합
			+ 각 cluster가 얼마나 조밀하게 모여 있는지를 나타냄
			+ 작을수록 잘 모여 있는 것
		+ k: cluster 수
		+ n: 전체 데이터 포인트 수
	+ CH Index가 클수록 cluster 내는 조밀하고, cluster 간은 잘 분리되어 있다는 것을 나타냄
		+ SSE (Sum of Squared Errors)는 cluster 내 응집도만 반영하지만 CH Index는 cluster 내 응집도와 cluster 간 분리도를 모두 고려하므로 SSE 보다는 조금 더 신뢰할 수 있는 clustering quality 지표임
	+ 결과 비교
		+ 기본 kmeans
			+ SSE: 95645.70359
			+ execution time: 187504.5317 ms
			+ Calinski–Harabasz Index: 15.62588368
		+ Hierarchical kmeans
			+ SSE: 95691.39094
			+ execution time: 118098.8245 ms
			+ Calinski–Harabasz Index: 13.95875857
	+ 최적의 K 찾기
		+ 또한 CH Index는 최적의 cluster 수(K)를 찾는 데 활용 가능
			+ SSE는 K가 커지면 항상 작아지므로 최적의 K를 찾는 기준이 될 수 없음
			+ 하지만 CH Index는 K가 커질 때 얻어지는 이득이 얼마나 의미있는 가를 판단할 수 있음 
				+ BCSS를 통해 cluster 간 분리도의 증가가 응집도 감소에 비해 충분히 큰지를 평가하기 때문
				+ 즉, 정말로 K를 늘려서 데이터를 더 나눈 것이 cluster끼리 유의미한 구분을 만든건지 판단
			+ https://www.mathworks.com/help/stats/clustering.evaluation.calinskiharabaszevaluation.html
			  ![[optimal-k-kmeans-result.png|250]] ![[optimal-k-find.png|250]]
#### 진행할 사항
+ 코드 실행 시 동일한 생성 데이터를 가지고 기본 kmeans와 hierarchical kmeans를 수행할 수 있도록 코드 수정 - 공정한 비교를 위함
	+ 현재는 매 코드 실행마다 샘플 데이터를 랜덤으로 생성하고 이후 kmeans 혹은 hierarchical kmeans를 선택적으로 진행하기 때문에 비교가 완전 공정하다고 보기 힘듦

#### Random Projection based Locality Sensitive Hashing (LSH)
* 보통의 가장 일반적인 초기화 K-Means++에 비해 고차원/대규모에서 현저히 계산량 감소
* 일반적인 LSH Coarse clustering
	* 고차원 데이터 (NxD)에 대하여 random vector (1D) 생성
	* 각 데이터 vector와 random vector의 dot product
	* 결과 값을 normalize하여 정수형으로 변환하여 bucket index로 만든다
	* Result
		* ![[lsh.png]]
		* 1개의 random vector를 사용한 결과 bucket imbalance 문제 발생
		* multi-probe tuning 필요
* Multi-bit LSH Coarse clustering
	* 기존 LSH기법은 projection 벡터 하나로 dot product
	* 여러 방향으로 투영하여 bit string 생성하여 정교한 hash code 생성
	* 더 세밀하게 구분이 가능하지만 너무 많이 구분하는 경우 bucket imbalance 발생 가능
	* 적합한 벡터 개수를 설정해야함
	* Result
		* ![[lsh_multibit.png]]
* Multi-probe LSH Coarse clustering
	* Multi-Probe LSH: Efficient Indexing for High-Dimensional Similarity Search (VLDB 2007)
	* Multi-probe는 _비슷한 해시코드로부터 여러 bucket 후보_를 고려해서 좀 더 균형 있게 분포시킴
	* i번째 비트를 반전시킨 새로운 해시 코드 생성하여 candidate bucket 생성
	* Result
		* ![[lsh_multiprobe.png]]

#### Random Projection based Multi-probe Locality Sensitive Hashing
* LSH coarse grouping 후 검증 
	* 각 bucket 내부의 평균거리를 계산하여 coarse grouping 평가
	* 각 bucket 내부에서 데이터 간 거리 간 거리가 작을수록 coarse clustering 이 잘된 것으로 판단
	* random grouping 방법으로 같이 확인
		* Result
		* ![[comparison.png]]
* Bucket avg distance comparison
	* ![[Pasted image 20250501123149.png]]

#### Plan
* 실험 dataset
	* 현재는 clustering 결과를 대략 확인하기 위한 synthetic data 사용
	* 중심점 주변으로 sampling 하여 dataset 생성
	* 주로 논문에서 사용하는 dataset으로 실험 예정

| Dataset&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Sample&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | Dimension&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |
| ------------------------------------- | ------------------------------------ | --------------------------------------- |
| RCV1                                  | 804,000                              | ~47,000                                 |
| SIFT1M                                | 1,000,000                            | 128                                     |
| GIST1M                                | 1,000,000                            | 960                                     |
| US Census 1990                        | 2,500,000                            | 68                                      |
|                                       |                                      |                                         |
* Fine clustering과 결합
	* weighted K-means
		* Bucket size-aware weighted k-means
* 전체 pipeline
	* 1단계: LSH-based Coarse clustering
		* Multi-bit 또는 Multi-probe LSH를 사용하여 데이터를 다수의 bucket으로 분할
	* 2단계: Bucket Size-Aware Weighted K-means 기반 Fine Clustering
		* 가중치가 높은 bucket에 더 많은 cluster 수 할당
		* 각 bucket에 대하여 weighted K-means 수행
	* pipeline
	* <img src="pipeline2.png" style="width:50%;">

### CUDA Memory Coalescing
reference: https://developer.nvidia.com/blog/how-access-global-memory-efficiently-cuda-c-kernels/
Device에서 프로세스가 실행되는 동안, threads는 32개의 threads 단위 warp로 finer grouping된다.
```
warp size
The warp size (effectively the SIMD width) of all current CUDA-capable GPUs is 32 threads.
```
#### Global Memory Coalescing
```
Arrays allocated in device memory are aligned to 256-byte memory segments by the CUDA driver.
```
CUDA driver는 global memory에 저장되는 배열을 256-byte 단위로 정렬한다. Device는 warp에서 32, 64, 128-byte단위의 transcation(aligned to their size)로 접근할 수 있다. warp의 threads는 aligned된 메모리 주소부터 읽으면 최소한의 transcation으로 구성할 수 있다. 
* CC 2.0 이상부터는 mis-aligned access여도 L1 cache로 load하여 유의미한 성능 차이는 없다.
#### Stride Memory Access
모든 compute capability 에서는 연속된 메모리(stride = 1)인 경우에 최적의 성능을 낸다. 연속적인 thread가 물리적으로 떨어진 메모리 주소에 접근할 때 성능 저하가 크게 일어난다. 
하지만, strided access가 필요한 경우가 있다. 이는 shared memory를 사용하여 처리할 수 있다.
shared memory는 thread block의 모든 thread가 공유하는 on-chip memory이다. global memory에서 coalesced fashion(연속 메모리 주소 access)으로  데이터를 shared memory로 load한 후, shared memory에서 strided access를 하면 성능 저하 없이 사용할 수 있다.


### 벡터 간 거리 계산 환경 세팅
- vector 간 cosine similarity 계산의 CUDA 기반 기본 코드 구현
- 100만 개, 1536 차원의 기준으로 측정했을 때 시간 측정이 오래 걸리기 때문에 단일 input vector와 나머지에 대해 비교하는 1 : N 거리 계산 구조

#### CUDA kernel 구조
- 각 thread는 dimension 만큼 loop
	- input vector와 thread index에 할당된 sample vector 간 내적과 norm 계산
	- 내적 값과 vector norm을 통해 cosine similarity 계산

#### 성능 분석
- 1 : N cosine similarity 계산
- local 2080 ti 환경에서 측정

| Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | <span style="color: yellow;">execution time (ms)</span>&#x20&#x20&#x20 |
| ------------------------- | -------------------------- | ---------------------------------------------------------------------- |
| 1M                        | 3                          | 0.303175                                                               |
| 1M                        | 768                        | 128.689557                                                             |
| 1M                        | 1536                       | 242.949283                                                             |
