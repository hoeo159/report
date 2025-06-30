#### 진행사항
* Linux Platform에서 진행을 위해 WSL2 에서 cuda-kmeans 환경 setting 완료
* Cuda-kmeans 확인을 위한 시각화 구현
* Cuda-kmeans의 GPU(2080, 3090, 4090) 성능 측정

#### Cuda-kmeans구성
* Update step
	* 각 cluster의 centroid를 새롭게 계산하는 작업. 각 클러스터에 속한 데이터 포인트의 평균을 구하여 새로운 centroid 계산
	* 각 클러스트에 속한 데이터 포인트의 좌표를 차원별로 합산하고 포인트 개수로 나눠 새로운 centroud 계산
	* atomic operation 사용하여 다중 스레드 환경에서 중심점 좌표 갱신. 여러 스레드가 동시에 동일한 메모리 업데이트 하는 것을 방지하고 효율적 업데이트 위한 CUDA 기반 병렬 알고리즘 사용

* Assignment Step
	* 각 데이터 포인트가 어떤 클러스터에 속하는지 결정. 
	* 각 데이터 포인트에 대해 Euclidean distance를 사용하여 모든 클러스터 중심점과의 거리 계산
	* 가장 가까운 cluster 찾은 후, 해당 포인트를 cluster에 할당
	* 1 iteration은 1update + 1assignment step

#### K-means clustering 시각화 결과

*다차원 데이터는 2차원에 표현이 불가능하기에 앞 2차원만 출력하도록 구현
k-means 클러스터링이 잘이루어지는지 확인하기 위해 data 차원을 2차원으로 설정

* Results
	* ### N = 512k, dim = 32, k = 128, iter = 1
	* dimension 32중 2차원만 출력하여 clustering 결과 확인 힘들다
	* ![[result_N512k_dim32_k128_iter1.png]]

    + ### N = 128k, dim = 2, k = 25, iter = 1
    + dimension을 2로 줄였지만 1 iteration이라 clustering이 잘되지 않음
    + ![[result_N128k_dim2_k25_iter1.png]]

    + ### N = 128k, dim = 2, k = 25, iter = 30
    + iteration 30번 수행했을 시 1개 cluster 제외 24개에 대해서 좋은 clustering 결과 확인
    + ![[result_N128k_dim2_k25_iter30.png]]

    + ### N = 128k, dim = 2, k = 25, iter = 1000
    + iteration 1000번 수행했을 시 비슷한 결과 확인
    + ![[result_N128k_dim2_k25_iter1000.png]]

    + ### N = 128k, dim = 2, k = 25, iter = 100000
    + iteration을 100000번 수행 시 안좋아진 결과 확인
    + ![[result_N128k_dim2_k25_iter100000.png]]
#### K-means clustering 성능 측정 결과

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | ---------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 512k                      | 32                         | 128 | 1                         | 14.9899                                                                |
|          NVIDIA 3090           | 512k                      | 32                         | 128 | 1                         | 2.968                                                                  |
|          NVIDIA 4090           | 512k                      | 32                         | 128 | 1                         |                                                                        |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | ---------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 128k                      | 2                          | 25  | 1                         | 0.539                                                                  |
|          NVIDIA 3090           | 128k                      | 2                          | 25  | 1                         | 0.1836                                                                 |
|          NVIDIA 4090           | 128k                      | 2                          | 25  | 1                         |                                                                        |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | ---------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 128k                      | 2                          | 25  | 30                        | 6.534                                                                  |
|          NVIDIA 3090           | 128k                      | 2                          | 25  | 30                        | 3.929                                                                  |
|          NVIDIA 4090           | 128k                      | 2                          | 25  | 30                        |                                                                        |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | ---------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 128k                      | 2                          | 25  | 1000                      | 184.83                                                                 |
|          NVIDIA 3090           | 128k                      | 2                          | 25  | 1000                      | 126.044                                                                |
|          NVIDIA 4090           | 128k                      | 2                          | 25  | 1000                      |                                                                        |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | ---------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 128k                      | 2                          | 25  | 100000                    | 15769.4                                                                |
|          NVIDIA 3090           | 128k                      | 2                          | 25  | 100000                    | 12231.2                                                                |
|          NVIDIA 4090           | 128k                      | 2                          | 25  | 100000                    |                                                                        |
#### Plan
* IVF 형식으로 변환
* 현재 data는 random으로 k-means 확인용 data generation 인데 디노티시아에서 사용하는 data의 형태로 변환
	* 이미지이면 이미지 차원에 맞게 data 넣기
* Load manager 구현
	* 1년차에 조금 구현해놓기
* 4090 측정
	* 4090 WSL. Cuda setting 후 execution time 측정