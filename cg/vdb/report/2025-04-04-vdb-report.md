### 1차년도 진행 사항
**Cuda K-means 구현 및 시각화 / 환경 setting
* Linux Platform에서 진행을 위해 WSL2 에서 cuda-kmeans 구현
* OpenAI embedding dataset (제안서 기준 high-dimensional dataset)
	* https://github.com/pgvectorBench/pgvectorBench -> 1536차원 500만개 데이터셋이 50만개씩 10개로 분할
	* parquet 형식을 cpp로 변환하여 사용할 수 있도록 setting 완료 및 성능 측정

**Optimization
* 커널에서 shared memory를 사용하여 d_centroid와 d_clust_size의 중복 접근 줄임
* Kernel parameter 최적화
	* Cuda API를 이용해서 device의 사양을 확인하고 최적의 parameter 값 정해줌
	* shared memory size 고려하여 chunksize및 TPB size 선정
* Shared memory bank conflict 해결
	* 기존에는 데이터를 chunk size로 나눠 shared memory에 올린 후 처리하는 방식
	* centroid의 개수가 커질 경우 모든 centroid를 shared memory에 담을 수 없어 다시 k를 나눠서 처리해야함
	* 또한 여러 thread가 동시에 같은 shared memory 주소에 접근하면서 bank conflict 발생
	* 차원을 쪼개지 않고, centroid를 쪼개서 넣으며 shared memory안에서 centroid 정보들을 훨씬 효율적으로 저장
* PCA k-means
	* 입력 데이터의 차원이 너무 커서 cuBLAS와 cuSOLVER라는 행렬 계산 라이브러리를 사용하여 cuda PCA 함수 구현
	* 100만 개 이상의 sample data는 좌표가 변하지 않기 때문에 한 번만 차원 축소한 값을 얻으면 계속 사용할 수 있고, (N >> K) K 개의 cluster centroid update 과정은 labeling 과정에서 cluster index만 사용하기 때문에 유효할 것이라 판단

**Data load manager 구현
* 처음에는 매 반복마다 전체데이터의 일부를 GPU로 복사하고 GPU에서 해당 데이터를 가지고 커널 실행
* 따라서 GPU는 커널 실행이 끝나기 전까지 다음 데이터를 준비할 수 없다
* 이를 해결하기 위해, GPU 메모리를 두개 영역으로 (2/N씩)으로 나누고,
	* 한 쪽 영역은 커널 실행에 사용
	* 다른 한 쪽 영역은 다음 batch 데이터를 업로드하는 2-way stream 방식 적용
* 디스크에서 데이터를 CPU 메모리로 로딩하는 과정에서 OS의 파일 I/O 최적화를 활용하기 위해 mmap을 사용하여 성능 높임

### 2차년도 진행 계획
**GPU 기반 벡터 간 거리 계산을 위한 GPU 기반 가속 기법 개발
* 다양한 query task에 대한 L2 Norm이나 Cosine Similarity와 같은 벡터 간 거리 계산을 가속하는 GPU 모듈 제작
* 가변적 batch size를 처리할 수 있게 하는 mini-batch 기반의 계산 모듈과 동시 요청되는 task들에 대한 데이터의 동적 GPU 로딩을 가속하는 memory load manager 개발
* 논문 작성 예정

