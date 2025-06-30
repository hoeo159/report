### 진행사항

#### 1536차원의 데이터 분할
+ GPU의 shared memory는 용량이 제한적-> 모든 데이터를 한번에 shared memory로 로드하는 것은 불가능
+ 기존의 shared memory 구현 방식은 shared memory의 용량이 한계에 다다른 경우 K-means clustering이 불가능
+ Clustering 결과 확인
	+ 기존의 code를 clustering하여 매 iteration마다 SSE (모든 point들의 각각 centroid 값 까지의 거리 합) 을 출력해본 결과 SSE 값이 커지는 문제 발생
	+ 이는 chunk로 나누었을 때 마지막 남는 부분이 0으로 초기화가 안되고 이전의 값이 들어가면서 발생
	+ 초기화를 진행한 결과 SSE 값이 정확히 줄어드는 것을 확인
+ Code
```
    extern __shared__ float shardMemory[];
    float* sharedPsum = shardMemory;
    int* sharedClusterSize = (int*)&sharedPsum[K * d_chunkSize];
```
	- shared memory 추가 할당
```
for (int offset = 0; offset < DIM; offset += chunkSize) {
	int curChunkSize = min(chunkSize, DIM - offset);
	for(int i = tid; i < K * curChunkSize; i += blockDim.x) {
		sharedPsum[i] = 0.0f;
	}
	for(int i = tid; i < K; i += blockDim.x) {
		sharedClusterSize[i] = 0;
	}
	__syncthreads();
	
	// update centroid
}
```
	- curChunkSize < chunkSize인 경우를 대비해 shared memory 초기화
	- sharedClusterSize : block 내에서 각 cluster가 가지는 data point 개수
	(local memory 역할)
```
for(int cur_iter = 1; cur_iter <= MAX_ITER; ++cur_iter)
{
	launch_kmeans_labeling(d_samples, d_clusterIndices, ... , dimension);
	// cudaDeviceSynchronize();

	launch_kmeans_update_center(d_samples, d_clusterIndices, ... , dimension);
	// cudaDeviceSynchronize();

	// cudaMemcpy(h_clusterIndices, ... , cudaMemcpyDeviceToHost);
	// cudaMemcpy(h_clusterCenters.data(), ... , cudaMemcpyDeviceToHost);
}
```
	- 불필요한 Memcpy 및 Synchronize(kmeans_dim에 한해서) 제거

+  Random dataset 결과 확인

| GPU         | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (s)</span>&#x20&#x20&#x20 |
| ----------- | ------------------------- | -------------------------- | --- | ------------------------- | --------------------------------------------------------------------- |
| NVIDIA 4090 | 512k                      | 32                         | 128 | 1000                      | 0.343                                                                 |
| NVIDIA 4090 | 1M                        | 1536                       | 25  | 10                        | 3.229                                                                 |
| NVIDIA 4090 | 1M                        | 1536                       | 25  | 100                       | 32.255                                                                |

#### OpenAI 임베딩 데이터셋
+ 제안서 기준 : openai_large_5m dataset
	+ nytimes보다 차원수가 6배 높고, 데이터셋 크기가 3배
	+ 데이터셋은 제안서 기준 100만개
	+ https://github.com/pgvectorBench/pgvectorBench -> 1536차원 500만개 데이터셋이 50만개씩 10개로 분할되어 있음
	+ iteration은 따로 나와있지 않아서 적당히 SSE값이 수렴하는 부분을 통해 확인
	+ Kmeans와 Shared Memory의 dimension chunking을 사용한 결과 비교
	+ open AI dataset 결과 확인

| GPU         | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (s)</span>&#x20&#x20&#x20 |
| ----------- | ------------------------- | -------------------------- | --- | ------------------------- | --------------------------------------------------------------------- |
| NVIDIA 4090 | 1M                        | 1536                       | 25  | 10                        | 3.256->1.626                                                          |
| NVIDIA 4090 | 1M                        | 1536                       | 25  | 36                        | 11.68->5.753                                                          |
|             |                           |                            |     |                           |                                                                       |



### Plan
+ 현재 shared memory에서 100만개 1536차원 기준으로 코드가 잘돌아감
+ 1억개, 10억개까지 datapoint를 늘리고, batch로 나누어 GPU에 로드하는 부분 구현중