### 2025 저널, 학회 후보
* Journal of Supercomputing
	* Journal Impact Factor: 2.5
	* Publisher: Springer
* IEEE Access
	* Journal Impact Factor: 3.4
	* Publisher: IEEE-INST ELECTRICAL ELECTRONICS ENGINEERS INC
* Computers & Electrical Engineering
	* Journal Impact Factor: 4.0
	* Publisher: PERGAMON-ELSEVIER SCIENCE LTD
* IEEE Transactions on Parallel and Distributed Systems (TPDS)
	* Journal Impact Factor: 5.6
	* Publisher: IEEE COMPUTER SOC
* Journal of Parallel and Distributed Computing (JPDC)
	* Journal Impact Factor: 3.4
	* Publisher: ACADEMIC PRESS INC ELSEVIER SCIENCE
* International Conference on Parallel Processing (ICPP)
	* conference
	* 최근 논문: Detailed Analysis and Optimization of CUDA K-means Algorithm (ICPP 2020)
* Engineering Science and Technology-An International Journal-JESTECH
	* Journal Impact Factor: 5.1
	* Publisher: ELSEVIER - DIVISION REED ELSEVIER INDIA PVT LTD

### 논문 진행 사항
#### 초기 grouping 기법
+  random sampling (기존 kmeans 방법)
	+  초기화가 제대로 되지 않으면 수렴이 매우 느리고 품질이 낮음
+  Kmeans++
	+  첫번째 중심점을 무작위로 선택하고 이를 기준으로 먼 거리에 있는 포인트를 위주로 선택
	+ 거리계산과 확률분포 샘플링을 여러번 해야해서 초기화 비용이 매우 큼
	+ 대규모 데이터셋에서 시간 소모가 매우큼
+ PCA
	+ PCA를 사용하여 dimension reduction
	+ 축소된 dimension에서 빠르게 K-means를 수행하고 원래 dimension으로 transform하여 초기화에 사용
	+ 대용량일 때 PCA 자체가 느릴 수 있다
+  LSH-based Initialization
	+  Locality-Sensitive Hashing을 사용하여 유사한 데이터 포인트들이 동일한 해시 버킷에 들어가도록 설계
	+  버킷 단위로 데이터를 그룹핑하여 빠르고 의미 있는 초기 클러스터 형성
	+  해시 함수 설계가 매우 중요
#### Hierarchical clustering 구현
+ 먼저 전체 데이터를 대상으로 작은 수의 K로 coarse clustering를 진행한 후, K개의 cluster 각각에서 다시 fine clustering을 진행하여 최종 clustering 결과 생성
```
// coarse clustering
for (int cur_iter = 1; cur_iter <= MAX_ITER/2; ++cur_iter) {
{
    launch_kmeans_labeling(d_samples, d_coarseIndices, d_coarseCentroids, N, TPB, k_coarse, DIM);
    cudaDeviceSynchronize();
    launch_kmeans_update_center(d_samples, d_coarseIndices, d_coarseCentroids, d_coarseSizes, N, TPB, k_coarse, DIM);
     cudaDeviceSynchronize();
     
     cudaMemcpy(h_coarseIndices.data(), d_coarseIndices, N * sizeof(int), cudaMemcpyDeviceToHost);
     cudaMemcpy(h_coarseCenters_out.data(), d_coarseCentroids, k_coarse * DIM * sizeof(float), cudaMemcpyDeviceToHost);
}
```

```
// fine clustering
#pragma omp parallel for schedule(dynamic)
for (int c = 0; c < k_coarse; c++) {
		// coarse cluster c에 속한 데이터의 인덱스를 모아서 indices에 저장
        std::vector<int> indices;
        for (int i = 0; i < N; i++) {
            if (h_coarseIndices[i] == c)
                indices.push_back(i);
        }
        int sub_N = indices.size();
        if (sub_N == 0) continue;

		// coarse cluster c에 해당하는 샘플만 h_subData에 복사
        std::vector<float> h_subData(sub_N * DIM);
        for (int i = 0; i < sub_N; i++) {
            int idx = indices[i];
            for (int d = 0; d < DIM; d++) {
                h_subData[i * DIM + d] = h_allSamples[idx * DIM + d];
            }
        }

		// 초기 fine 중심점 설정
        std::vector<float> h_subCentroids(k_fine * DIM);
        for (int j = 0; j < k_fine; j++) {
            int idx = j % sub_N;
            for (int d = 0; d < DIM; d++) {
                h_subCentroids[j * DIM + d] = h_subData[idx * DIM + d];
            }
        }

		// CUDA 메모리 할당 진행
		...

		// fine kmeans 반복 수행
		for (int cur_iter = 1; cur_iter <= MAX_ITER/2; ++cur_iter) {
			launch_kmeans_labeling(d_subData, d_subIndices, d_subCentroids, sub_N, TPB, k_fine, DIM);
			cudaDeviceSynchronize();
			launch_kmeans_update_center(d_subData, d_subIndices, d_subCentroids, d_subSizes, sub_N, TPB, k_fine, DIM);
			cudaDeviceSynchronize();
		}

		// 결과 저장
		...
```

+ e.g., original kmeans에서 K = 25, Iter = 10이라면, hierarchical clustering에선 K_coarse = 5, Iter_coarse = 5, K_fine = 5, Iter_fine = 5로 설정하여 clustering 진행 -> 총 cluster의 개수는 같도록 함
	+ cluster 중심이 수렴하면 clustering이 자동으로 종료되도록 코드 수정 예정
+ OpenMP API를 통해 fine clustering 병렬로 수행
##### 실험 결과
+ K = 25, Iteration = 10, Num = 100000, Dim = 1536
	+ Original kmeans
	  ![[result-standard-N100K-dim1536-k25-iter10.png|300]]
	+ Hierarchical kmeans
	  ![[result-hierarchical-N100K-dim1536-kc5-kf5-iter10.png|300]]
+ K = 2500, Iteration = 10, Num = 100000, Dim = 1536
	+ Original kmeans
	  ![[result-standard-N100K-dim1536-k2500-iter10.png|300]]
	  ![[result-standard-N100K-dim1536-k2500-iter2.png|300]]
	+ Hierarchical kmeans
	  ![[result-hierarchical-N100K-dim1536-kc50-kf50-iter10.png|300]]
##### 진행할 사항
+ cluster 중심이 수렴하면 clustering 종료되도록 코드 수정
+ SSE 외에 추가적인 clustering 평가 지표 도입 (e.g., Silhouette 지표, Dunn Index)
#### Memory Coalescing
CUDA memory access는 32개 thread로 이루어진 Warp 단위로 발생. 이 때 Cache Line 연속된 메모리 공간에 access하는 것을 Memory Coalescing이라함.
Warp 내 모든 thread들의 access가 32byte 내에서 이뤄진다고 하면 하나의 transection으로 access가능
```
            for (int i = tid; i < DIM; i += blockDim.x)
            {
                float dataPoint = d_samples[blockIdx.x * DIM + i];
                float kPoint = sharedCentroids[j * DIM + i];
                float diff = dataPoint - kPoint;
                squaredDistanceMem[i] = diff * diff;
            }
```
각 thread가 연속된 메모리 영역을 access하고 있으므로, memory coalescing을 만족한다고 볼 수 있음.




