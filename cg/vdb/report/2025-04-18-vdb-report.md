### 논문 진행 사항
#### cluster 중심 수렴 후 clustering 자동 종료
+ 기존 다른 라이브러리들이 사용하는 threshold
	+ SciPy 라이브러리 - 기본값 1e-05 - https://docs.scipy.org/doc/scipy-1.15.2/reference/generated/scipy.cluster.vq.kmeans.html
	+ Oracle Machine Learning (OML4Py) - 기본값 1e-03 - https://docs.oracle.com/en/database/oracle/machine-learning/oml4py/1/mlpug/k-means.html
+ 사용한 threshold 값
	+ 기본 kmeans - 1e-04
	  ![[convergence-kmeans.png|300]]
	+ hierarchical kmeans
		+ coarse - 0.05: 전체 데이터를 대략적으로 분할하는 것이 목적이므로, 큰 threshold 값을 사용해 전반적인 분포 파악만을 진행한 후 fine clustering 단계로 넘어가도록 하기위해 설정
		  ![[convergence-hierarchical-coarse.png|400]]
		+ fine - 1e-05: coarse 단계에서 분할된 cluster 안에서 정밀한 clustering을 수행하는 것이 목적이므로, 작은 threshold 값을 사용해 좀 더 엄격한 수렴 조건을 설정
		  ![[convergence-hierarchical-fine.png|450]]
##### 진행할 사항
+ fine clustering CUDA 통해서 병렬로 처리
### Related work 조사
*Kmeans 관련
- [ ] k-means++: The Advantages of Careful Seeding
- [ ] Yinyang K-Means: A Drop-In Replacement of the Classic K-Means with Consistent Speedup
- [ ] Efficient and Scalable k-Means on GPUs
- [ ] Detailed Analysis and Optimization of CUDA K-means Algorithm
*Mini-batch 관련
- [ ] Web-Scale K-Means Clustering
- [ ] Nested Mini-Batch K-Means

### 논문 진행사항
![[model.png]]

* 모델 설계
	* CUDA-based LSH로 초기 coarse grouping
	* Bucket안의 point 개수에 따라 weight를 줘서 fine K-means 진행
	* cluster 중심 수렴 할 때까지 K-means clustering

