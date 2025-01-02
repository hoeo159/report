
- code
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
