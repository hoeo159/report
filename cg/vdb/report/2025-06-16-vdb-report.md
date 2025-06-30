### 2차년도 진행 상황
#### Load manager	
+ asynchronous transfer and execute
	+ Async 연산을 하기 위해서는 data를 pinned memory로 고정시켜야 함
	+ stream을 # of total data / batch_size 만큼 생성 후, stream 마다 아래 코드를 실행
```
streams = cudaStreamCreate <- (1M data / batch_size)
for i in streams
	memcpy                       // pageable -> pinned  (input)
	cudaMemcpyAsync              // pinned -> device    (input)
	launch_cosine_similarity     // launch kernel
	cudaMemcpyAsync              // device -> pinned    (output)
	memcpy                       // pinned -> pageable  (output)
```
- 현재 kernel code는 overlap을 확인하였으나 모든 커널의 시작과 끝 전후로 메모리 전송이 지연됨
- 측정 결과 (2080 ti, 10회 측정 후 평균), nvprof 결과 순서대로 init, async, async+pinned

|              | execution time(ms) |
| :----------- | :----------------: |
| init         |      1727.990      |
| async        |      1521.064      |
| async+pinned |      1538.222      |
![[0605.png]]
##### Todo
1. 

#### 거리 계산 가속
+ Grouping Weighted clustering을 통한 거리 계산 가속
+ L1, L2 norm 계산 모듈 제작
	```
	__device__ float get_distance_l1(const float* point, const float* centroid, int DIM)
	{
		float dist = 0.0f;
		for (int d = 0; d < DIM; ++d)
			dist += fabsf(point[d] - centroid[d]);
		return dist;
	}

	__device__ float get_distance_l2(const float* point, const float* centroid, int DIM)
	{
		float squaredDistance = 0.0f;
		for (int d = 0; d < DIM; ++d) {
			float vecDiff = point[d] - centroid[d];
			squaredDistance += vecDiff * vecDiff;
		}
		return sqrtf(squaredDistance);
	}
	
	__device__ float get_distance(const float* point, const float* centroid, int DIM, NormType normType)
	{
		if (normType == L1_NORM) return get_distance_l1(point, centroid, DIM);
		else return get_distance_l2(point, centroid, DIM);
	}
	```

##### Todo
1. random labeled vector data 생성 코드 개발
2. Cosine similarity 계산 모듈 제작
3. PCA 차원 축소를 통한 거리 계산 가속


### FastK 진행 상황
#### Hierarchical K-means
+ g_init rule 제작
	+ g_init = alpha x K 
		+ alpha: grouping 비율
			+ 관련 연구, 문서
				+ 2로 설정
					+ NVIDIA RAPIDS cuVS 라이브러리 https://docs.rapids.ai/api/cuvs/stable/cpp_api/cluster_kmeans/
				+ 3으로 설정: 
					+ Riz, Luigi, et al. "Novel class discovery for 3d point cloud semantic segmentation." _Proceedings of the IEEE/CVF conference on computer vision and pattern recognition_. 2023. -> 실험적으로 3이란 값을 설정한 것으로 보임
		+ K: 최종 cluster 수
##### Todo
1. g_init rule 보완 -> 전체 데이터 개수도 고려하도록

#### Hierarchical K-means
* 기존 방식
	* 초기 grouping을 5k의 group으로 K-means 사용하여 초기 clustering
	* Level 별 iteration 및 runtime 측정
	* 기준 Data: 100K TPB 256 K 3 split_num 3 max iter 1000 dimension 100
		* level 개수에 따른 runtime
			* Standard K-means Time: 1718 ms
			* Hierarchical K-means
				* 1 level: 750 ms
				* 2 level: 952 ms
				* 3 level: 915 ms
				* 4 level: 1054 ms
				* 5 level: 873 ms

	* Level에 따른 iteration 및 runtime 측정
		* ![[level_iteration.png|400]]
		* level 별 iteration을 측정해본 결과 이미 초기 stage의 kmeans coarse clustering에서 대부분 이미 convergence 완료
* LSH coarse clustering 사용
* Pipepline
	* ![[HW_pipeline.png]]
	* kmeans coarse clustering처럼 초기에 이미 bucket에 정확하게 clustering되서 들어감
	* ![[LSH2.png|400]]
		* Synthetic dataset 수정-분포 조정
		* ![[LSH1.png|400]]
		* **Multi-projection LSH 사용**: 다양한 방향에서 hash하고 다수결 또는 평균 해시값으로 bucket 할당