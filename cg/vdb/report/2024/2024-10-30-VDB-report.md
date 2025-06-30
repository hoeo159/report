#### 진행사항
* Linux Platform에서 진행을 위해 WSL2 에서 cuda-kmeans 환경 setting 완료
* Cuda-kmeans 확인을 위한 시각화 구현
* Cuda-kmeans의 GPU(2080, 3090, 4090) 성능 측정

#### Cuda-kmeans구성
### kmeans.cu
* kMeansClusterAssignmentKernel
	* 각 스레드는 하나의 데이터 포인트를 처리하며 데이터 포인트마다 모든 중심까지의 거리를 계산하여 가장 가까운 중심찾고 클러스터 번호를 d_clust_assn에 할당
* kMeansCentroidUpdateKernel
	* atomicAdd를 사용하여 centroid배열에 클러스터에 속하는 데이터 포인트 좌표의 누적합 계산
* kMeansCentroidAverageKernel
	* 누적합을 계산한 각 중심 좌표를 클러스터의 크기로 나눠 최종 평균 좌표(새로운 중심 위치)를 계산
* launchKMeansClusterAssignment, launchKMeansCentroidUpdate
	* 커널 함수 실행
### kmeans.cpp
* generate_clustered_data
	* kmeans 결과 확인용 random data point 생성
* main
	* parameter 입력 받아 GPU 메모리 할당 후 데이터 생성 및 복사
	* roop 실행하며 kmeans 알고리즘 수행
	* 중심 좌표와 각 데이터 포인트의 클러스터 할당 정보를 txt 파일로 저장 후 python kmeans 시각화에 활용

#### K-means clustering 시각화 결과
![](results/vis.png)

#### K-means clustering 성능 측정 결과

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: blue;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | -------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 512k                      | 32                         | 128 | 1                         | 55.3646                                                              |
|          NVIDIA 3090           | 512k                      | 32                         | 128 | 1                         |                                                                      |
|          NVIDIA 4090           | 512k                      | 32                         | 128 | 1                         | 10.5712                                                              |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: blue;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | -------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 128k                      | 2                          | 25  | 1                         | 7.669                                                                |
|          NVIDIA 3090           | 128k                      | 2                          | 25  | 1                         |                                                                      |
|          NVIDIA 4090           | 128k                      | 2                          | 25  | 1                         | 2.6313                                                               |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: blue;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | -------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 128k                      | 2                          | 25  | 30                        | 74.4889                                                              |
|          NVIDIA 3090           | 128k                      | 2                          | 25  | 30                        |                                                                      |
|          NVIDIA 4090           | 128k                      | 2                          | 25  | 30                        | 10.8775                                                              |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: blue;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | -------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 128k                      | 2                          | 25  | 1000                      | 2487.77                                                              |
|          NVIDIA 3090           | 128k                      | 2                          | 25  | 1000                      |                                                                      |
|          NVIDIA 4090           | 128k                      | 2                          | 25  | 1000                      | 291.152                                                              |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: blue;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | -------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 128k                      | 2                          | 25  | 100000                    | 251292                                                               |
|          NVIDIA 3090           | 128k                      | 2                          | 25  | 100000                    |                                                                      |
|          NVIDIA 4090           | 128k                      | 2                          | 25  | 100000                    | 25031                                                                |
#### Plan
* 현재는 기본적인 Cuda K-means 구현한 상태라, 알고리즘 최적화를 통해 execution time 줄이기
* 1M data를 1000 iteration 수행해본 결과, 5550ms -> 더많은 data를 위한 data load manager 개발