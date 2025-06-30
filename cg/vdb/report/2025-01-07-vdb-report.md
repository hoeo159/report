### 진행사항
#### 테스트를 위한 labeled dataset search
+ 현재 쓰고 있는 openai 데이터셋은 labeling이 되어 있지 않아 정확한 clustering 정확도를 알 수 없음
	+ SSE로 판단하고 있었으나, 이것 또한 정확하다 볼 수 없음
+ 따라서 labeling 된 데이터셋으로 코드 실행 결과 분석이 필요
+ 찾은 데이터셋
	+ Fish Dataset - csv
		+ 4080개의 데이터
		+ 3차원 featurelength, weight, w_l_ratio)와 label(species 8개)로 구성
	+ Star Dataset - csv
		+ 100,000개의 데이터
		+ 17 feature columns과 1 class column으로 구성
			+ class: 3개 (star, galaxy, quasar)
	+ Human Activity Recognition Using Smartphones Dataset - txt
		+ 7352(train) + 2947(test) 개의 데이터
		+ 561 feature
		+ class: 6개 (WALKING, WALKING_UPSTAIRS, WALKING_DOWNSTAIRS, SITTING, STANDING, LAYING)
	+ Forest Cover Type Dataset - data
		+ 581,012개의 데이터
		+ 54 feature, 1 class
			+ class: 7개
### Plan
+ 테스트용 데이터셋 결정 및 해당 데이터셋 사용 코드 구현


#### Kernel parameter 최적화
- 현재 clusterSize와 TPB(Threads per Block) 값을 특정 상수로 직접 입력해서 진행 중.
- Cuda API를 이용해서 device의 사양을 확인하고 최적의 parameter 값을 정해주어야 함.
	- Cuda Device 사양 확인
		- `cudaDeviceProp prop; cudaGetDeviceProperties(&prop, 0);` 
		-  device global memory, shared memory, maxTPB, warp size etc...
- parameter 값 선정 기준
	- shared memory size = (K * dimension * sizeof(float)) 이며 chunkSize는 sharedmemory의 공간을 넘을 수 없고 또 너무 작으면 성능이 하락. 대략 32(warp size) 부터 소요 시간이 비슷할 것이라 추측
	- data = 100000, K = 100, iter = 100, D = 1536 일 때 chunkSize에 따른 소요 시간
	- ![[clusterSize(K=100).png||500]]
	- chunkSize 후보군
		- shared memory를 K * sizeof(float) 만큼 나눈 maximum을 chunkSize 선정
		- chunkSize limit에서 적당한 작은 수 $\epsilon$을 빼서 최대한 shared memory 채워서 사용
	- TPB는 chunkSize를 넘지 않는 pow of 2를 선정. device의 maxTPB에 가깝게 꽉 채웠을 때 오히려 시간이 크게 나왔기 때문에 maxTPB / 2 값을 TPB limit로 정함
	- data = 100000, K = 100, iter = 100, D = 1536 일 때 TPB에 따른 소요 시간 (chunkSize = 96)
	  ![[TPB(K=100).png|500]]


#### Mini batch 구현
* 현재 데이터셋이 일정 이상일 때, cpu와 gpu의 메인 메모리 크기를 넘어가는 문제가 있음. 
* 데이터셋 N개를 인자로 들어온 n개씩 쪼개 n개의 데이터씩 gpu 메모리에 올리고 처리하는 방식을 루프를 돌며 진행함.
```
for (int i = 0; i < N; i += n)
{
    n = std::min(maxPartialN, N - i);
    cudaMemcpy(d_samples, h_samples.data() + (i * dimension), n * dimension * sizeof(float), cudaMemcpyHostToDevice);
    launch_kmeans_labeling(d_samples, d_clusterIndices, d_clusterCenters, n, i, TPB, K, dimension);
    cudaDeviceSynchronize();
}

n = maxPartialN;
for (int i = 0; i < N; i += n)
{
    n = std::min(maxPartialN, N - i);
    cudaMemcpy(d_samples, h_samples.data() + (i * dimension), n * dimension * sizeof(float), cudaMemcpyHostToDevice);
    launch_kmeans_update_center(d_samples, d_clusterIndices, d_clusterCenters, d_clusterSizes, n, i, TPB, K, dimension);
    cudaDeviceSynchronize();
}
```
* cpu의 메인 메모리의 데이터셋을 쪼개서 gpu에 올리는 과정만 구현했기 때문에 추가적인 구현이 필요함.
### Plan
* 데이터셋이 1000만개를 넘어간다면, cpu 메인메모리에 모든 데이터를 올리는 것도 불가능하기 때문에, .csv파일과 같이 디스크에 있는 데이터를 쪼개서 읽어 처리하고 다시 파일에 쓰도록 구현
* 데이터를 쪼개서 host to device로 데이터를 전송하는데 큰 비용이 들기 때문에, gpu 프로세싱과 cpu에서의 데이터 업로드를 병렬로 진행하는 stream 방식으로 구현
### Result
* TPB 측정

| GPU         | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | TPB | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (s)</span>&#x20&#x20&#x20 |
| ----------- | ------------------------- | -------------------------- | --- | --- | ------------------------- | --------------------------------------------------------------------- |
| NVIDIA 4090 | 1M                        | 1536                       | 32  | 25  | 10                        | 3.221                                                                 |
| NVIDIA 4090 | 1M                        | 1536                       | 256 | 25  | 10                        | 2.279                                                                 |
| NVIDIA 4090 | 1M                        | 1536                       | 32  | 25  | 36                        | 11.592                                                                |
| NVIDIA 4090 | 1M                        | 1536                       | 256 | 25  | 36                        | 8.125                                                                 |

* mini-batch 측정

| GPU         | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | batch  | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (s)</span>&#x20&#x20&#x20 |
| ----------- | ------------------------- | -------------------------- | ------ | --- | ------------------------- | --------------------------------------------------------------------- |
| NVIDIA 4090 | 1M                        | 1536                       | x      | 25  | 10                        | 3.221                                                                 |
| NVIDIA 4090 | 1M                        | 1536                       | 100000 | 25  | 10                        | 51.967                                                                |
| NVIDIA 4090 | 1M                        | 1536                       | 10000  | 25  | 10                        | 55.575                                                                |
| NVIDIA 4090 | 1M                        | 1536                       | 1000   | 25  | 10                        | 169.154                                                               |

### Plan
- K cluster 확장
	- chunkSize가 warp size인 32까지 놓게 된다면 최대 372개의 K까지 에러는 없음 
	- 만약 1000개 이상의 K가 생길 때 분할 해서 clustering이 진행하도록 설계해야 함
