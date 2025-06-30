### 논문 진행 사항
#### Calinski–Harabasz Index 사용 논문
+ Caliński, Tadeusz, and Jerzy Harabasz. "A dendrite method for cluster analysis." _Communications in Statistics-theory and Methods_ 3.1 (1974): 1-27.
	+ Calinski–Harabasz Index를 처음으로 소개한 연구, clustering의 유효성을 평가하기 위한 새로운 방법을 제안
+ Maulik, Ujjwal, and Sanghamitra Bandyopadhyay. "Performance evaluation of some clustering algorithms and validity indices." _IEEE Transactions on pattern analysis and machine intelligence_ 24.12 (2003): 1650-1654.
	+ 다양한 clustering 알고리즘과 유효성 지표를 비교 평가
+ Khan, Iliyas Karim, et al. "Determining the optimal number of clusters by Enhanced Gap Statistic in K-mean algorithm." _Egyptian Informatics Journal_ 27 (2024): 100504.
	+ CH Index를 활용하여 최적의 k 수 찾기
+ Hasan, Fatima M., et al. "Enhanced unsupervised feature selection method using crow search algorithm and Calinski-Harabasz." _International Journal of Computational Methods and Experimental Measurements_ 12.2 (2024): 185-190.
+ Yang, Kui, et al. "Classification and evaluation of driving behavior safety levels: A driving simulation study." _IEEE Open Journal of Intelligent Transportation Systems_ 3 (2022): 111-125.
	+ 평가 지표로 활용
+ Sinaga, Kristina P., and Miin-Shen Yang. "Unsupervised K-means clustering algorithm." _IEEE access_ 8 (2020): 80716-80727.
	+ 평가 지표로 활용
#### Hierarchical K-means code 수정
1. 전체 데이터에 대하여 10개의 Group이 생기도록 1차 grouping 진행 ![[first-grouping.png|450]]
2. 10개의 group을 다시 각각 10개의 group으로 나눠서 총 100개의 group 생성 (압축해제) ![[first-unzip.png|450]]
3. 100개의 group에 대해 clustering (K_final=10) 진행 ![[first-clustering.png|450]]
4. 100개의 group을 다시 각각 10개의 group으로 나눠서 총 1000개의 group 생성 (압축해제)
5. 1000개의 group에 대해 clustering (K_fianl=10) 진행
6. 위 과정(압축해제->clustering)을 원하는 만큼(reps) 반복
7. 최종 clustering: 마지막 압축해제된 모든 그룹들을 K_final(여기에선 K=10) 값으로 K-means 수행하여 K_final 개의 cluster 생성
+ 효과
	+ 이를 통해 상위 단계 clustering에서 이상치(noise)가 모여 있던 cluster는 하위 단계에서 다시 sub-group으로 나눠지고 다시 clustering됨으로써 더욱 정확한 clustering이 가능해짐
	+ 대규모 데이터 전체를 한 번에 clustering 하는 것보다 group으로 묶어 clustering을 진행하기 때문에 거리 연산 횟수가 줄어들어 더 빠른 clustering이 가능해짐
	+ 앞선 과정으로 점점 정확히 업데이트 되어가는 centroids를 기반으로 sub-group clustering이 이뤄지기 때문에 더 빠른 수렴 가능
#### 공정한 비교를 위한 Data Generation 코드 수정 - Seed 고정
+ 모든 비교 실험에 대해 동일한 seed로 생성된 데이터 포인트 사용
#### Hierarchical K-means 구현 사항
* Idea
	* Case: 1000개의 dataset을 K=3으로 clustering
	* 전체 데이터에 대해 먼저 10개의 group이 생기도록 coarse clustering
		* coarse clustering은 현재 LSH기반으로 빠르게 grouping 하도록 구현
	* 10개의 group을 group끼리 K-means clustering
		* 10개의 group의 centroid 먼저 계산
		* 그룹별로 누적된 좌표합을 포인트 개수로 나눠서 centroid 계산 -> mean 사용
		* centroid끼리의 K-means clustering
	* group 압축해제
		* 10개씩 10개로 압축해제
		* 전체 100개의 subgroup centroid에 대해 다시 k=3 K-means 수행
	* Final full data K-means
		* 최종적으로 1000개의 data에 대해 k=3 K-means 수행
		* 이미 잘 나뉘어진 cluster 구조를 기반으로 하기에, 더 적은 iteration으로 수렴 가능
* 구현 사항
	* LSH기반 coarse clustering 후 group별 K-means clustering vs K-means기반 coarse clustering

K_coarse = 25
group_coarse = 3

| N&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | LSH coarse clustering&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; | K-means coarse clustering&nbsp;&nbsp;&nbsp;&nbsp;&nbsp; |
| ------------------------------- | --------------------------------------------------- | ------------------------------------------------------- |
| 1000                            | 1964.15 ms                                          | 2371.58 ms                                              |
| 10000                           | 2377.44 ms                                          | 2464.87 ms                                              |
| 100000                          | 2840.31 ms                                          | 7120.63 ms                                              |
|                                 |                                                     |                                                         |


### 2차년도: 효율적인 벡터 간 거리 계산을 위한 GPU 가속 기법 개발

#### Mini-batch 기반의 가변적 batch size 처리 모듈 개발
- 목표
	- N = 1,000,000, Dimension = 1536의 vector data에 대해 각 100,000개, 10개 mini-batch를 통한 비동기 처리
- 구현 내용
	- mini-batch size에 따른 `cudaStream_t` 배열 생성
	- cpu : `cudaMemcpyAsync`를 통해 host -> device로 memory copy
	  gpu : 거리 계산 커널 코드 실행
	- cpu와 gpu는 비동기 처리
- 결과
	- original code : 1699.53606 ms
	- async code : 1396.840606 ms
- 진행 계획
	- pinned memory로 data 잡고 계산하기
	- 가변적 batch_size 처리(실시간 요청 task 처리)