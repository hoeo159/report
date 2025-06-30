### 진행사항
#### Shared memory bank conflict 해결
* 기존의 chunk size로 나눠서 shared memory에 담고 실행하는 방식은 K(centroid의 개수)가 늘어날 때, shared memory에 모두 들어가지 않아 K를 또 다시 나눠야 하는 문제를 가짐
* Thread 한 개가 최대 1536차원의 dimension을 순차적으로 계산하며, 동시에 여러 thread가 같은 shared memory주소에 접근하기 때문에 bank conflict 발생
* dimension을 쪼개는 것이 아니라, 하나의 centroid의 전체 dimension을 모두 넣는 대신, K를 쪼개서 shared memory에 넣는 방식을 사용함.(최대 shared memory에 1536차원 기준 8개의 centroid가 들어감)
* 하나의 thread가 아닌 하나의 thread block이 데이터 포인트 한 개를 담당하도록 하여, 각각의 thread는 하나의 데이터 포인트의 각각의 dimension을 병렬적으로 처리하도록 함.
* 각각의 dimension은 shared memory의 순차적인 영역을 접근하기 때문에 bank conflict문제를 해결할 수 있음.
* 32개의 순차적인 thread가 순차적으로 데이터를 접근하도록 함.
#### Plan
+ 최대 들어갈 수 있는 shared memory의 centroid 개수를 하드코딩하였기 때문에 Cuda API를 이용해 device의 사양에 맞게 할당하도록 해야됨

| GPU         | Datapoint &#x20&#x20&#x20 | Dimension &#x20&#x20&#x20 | TPB | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (s)</span>&#x20&#x20&#x20 |
| ----------- | ------------------------- | ------------------------- | --- | --- | ------------------------- | --------------------------------------------------------------------- |
| NVIDIA 4090 | 1M                        | 1536                      | 32  | 25  | 10                        | 3.221->9.416                                                          |
| NVIDIA 4090 | 1M                        | 1536                      | 256 | 25  | 10                        | 2.279->1.928                                                          |
| NVIDIA 4090 | 1M                        | 1536                      | 32  | 25  | 36                        | 11.592->33.982                                                        |
| NVIDIA 4090 | 1M                        | 1536                      | 256 | 25  | 36                        | 8.125->6.927                                                          |
#### Mini batch Stream 방식으로 변경
* 기존의 mini batch는 host to device 메모리 할당과 kernel 실행을 동기적으로 하기 때문에 속도가 느리다.
* 읽어서 gpu에 올려야 하는 데이터셋은 데이터 포인트 샘플이다.
* 데이터 포인트는 kernel에서 지속적으로 사용되기 때문에 다음 데이터 셋을 gpu에 올리면서 kernel을 동시에 실행하기 어려움
* 한 번에 N개의 데이터를 gpu에 올릴 수 있을 때, N/2개의 두 개의 영역으로 나눠, 동시에 한 쪽 영역은 gpu에 데이터를 올리고, 나머지 업로드된 영역은 kernel 실행에서 사용된다.
* 2-way stream 방식처럼 사용하고자 함.
#### Mini batch data stream 구현
* 기존에는 전체 샘플 데이터를 gpu에 올리는 방식을 사용했기 때문에 데이터셋이 천만건에 가까워지면 cpu memory allocation 실패가 되었다. 이를 해결하기 위해 디스크 파일에 있는 데이터를 읽어 일부분씩만 cpu와 gpu에 올려서 처리하는 방식을 사용했다.
* cpu에 올릴 수 있는 chunk size로 나눠서 gpu에 읽는 기본적인 방식은 cpu와 gpu가 동기적으로 작동하기 때문에 cpu to device latency로 인해 처리 속도가 매우 느려진다.
* 이를 해결하기 위해, cpu to device 영역을 두 개로 나눈 후, 한 쪽 영역에서는 cpu to device 작업(cpu 작업)을 하고, 나머지 영역은 커널 작업(gpu 작업)을 진행해서, 어느정도 비동기적으로, 겹쳐서 작업할 수 있도록 했다.
* 데이터셋을 읽어서 cpu메모리로 올리는 과정은 직관적이고, os의 최적화 도움을 받기 위해 mmap을 활용했다.
```
float *mappedSample = (float *)mmap(NULL, N * sizeof(float) * dimension, PROT_READ, MAP_PRIVATE, sampleFd, 0); //sample data file mmap
```

```
for (int j = 0; j < 2; j++)
{
    int offset = i * CHUNKSIZE + j * maxn;
    if (offset >= N)
        break;
    int n = std::min(maxn, N - offset);
    memcpy(h_pinned[j], &mappedSample[offset * dimension], n * sizeof(float) * dimension);
    cudaMemcpyAsync(d_samples_div[j], h_pinned[j], n * dimension * sizeof(float), cudaMemcpyHostToDevice, streams[j]);
    launch_kmeans_labeling(d_samples_div[j], d_clusterIndices, d_clusterCenters, d_clusterSizes, n, offset, TPB, K, dimension, streams[j]);
}
```
#### Plan
+ 디스크의 데이터를 cpu를 거쳐서 gpu에 업로드하기 때문에 DMA같이 직접적으로 gpu에 데이터를 올릴 수 있는 방법을 찾아보고자 한다.
#### kmeans validation을 위한 test code 구현
+ 사용한 데이터셋
	+ Fish_data.csv
		+ 4080개의 데이터, label(species 9개)과 3차원 feature(length, weight, w_l_ratio)로 구성
+ 테스트 커맨드
	+ ./kmeans_test fish_data.csv 128 9 100
		+ 실행파일, 데이터셋, TPB, K, ITER
+ 실행 결과![[fish-data-clustering-result.png]]
#### Plan
+ cluster center 사이의 거리 비교 코드 작성 
	+ 동일한 종이 여러 클러스터에 분포하는 경우, 클러스터링이 올바르게 수행되었는지 검증하기 위해
	+ 가까운 클러스터에 동일한 종이 존재하면 비교적 정상적인 결과로 간주
+ 다른 데이터셋에서도 정상적으로 작동하도록 코드 수정

#### PCA K-means(Plan)

- D차원을 D' 차원으로 줄이기 위해 cublas와 cusolverDn 행렬 계산 라이브러리를 활용해서 cuda PCA 함수 제작
- 100만 개 이상의 sample data는 좌표가 변하지 않기 때문에 한 번만 차원 축소한 값을 얻으면 계속 사용할 수 있고, (N >> K) K 개의 cluster centroid update 과정은 labeling 과정에서 cluster index만 사용하기 때문에 유효할 것이라 판단
- for 문 내부에서 update 된 cluster centroid를 다시 PCA해서 변환해주는 과정과 D차원 그대로 kmeans를 했을 때 속도 차이를 비교해야 할 필요가 있다
	- 차원 축소된 sample과 centroid 만 이용해서 k-means 진행
	- 현재 7900대에 SSE가 수렴하는 local minimum 문제

- Fisher_data
  ![[pca_1.png|500]]  ![[pca_3.png]]
- random_data
  ![[pca_2.png|500]]