2차년도 계획 작성
1. CUDA stream을 통한 비동기 처리 -> 민호
2. Mini batch -> 민호

전체적으로 뭘 할지, 왜 하는지, 어떤 효과가 있을 것 같은지, 구체적으로 어떤식으로 할건지

- CUDA stream을 통한 비동기 처리
	- 목적
		- 거리 계산 모듈과 동시 요청되는 task들에 대한 데이터의 동적 GPU loading을 가속화
	- 기대 효과
		- 연산과 전송이 동시에 진행되어 전체 처리 시간이 감소하고, pipeline 처리 성능 향상
	- 개발 계획
		- CUDA stream을 사용하여 커널 실행과 메모리 복사를 비동기적으로 실행
		- 각 batch에 대한 CUDA stream 생성, `cudaMemcpyAsync`와 kernel launch를 command queue에 enqueue한 뒤, stream 동기화를 수행
- Mini batch
	- 목적
		- 전체 데이터셋을 가변적 크기 batch size로 나누어 task query로 전달
	- 기대 효과
		- 그래프 빌드에서 매 요청마다 필요한 데이터셋의 크기가 달라지는 가변적 batch size 처리
	- 개발 계획
		- input size가 다른 task query들에 대해서도 비동기적으로 대응하고, GPU 메모리 사용을 최적화
		- 입력 크기에 따라 batch size를 dynamic하게 설정, 각 batch에 대한 stream을 비동기적으로 GPU에 전송하여 연산을 수행하며, memory loader는 요청 단위로 필요한 데이터만 선별적으로 loading

- 1차년도
    - K-means vector clustering
        
        - Out-of-core 데이터셋의 GPU K-means clustering 성능 측정
    - 대규모 데이터셋 동적 로딩 기법
        - GPU Cache 매니저
    - GPU-CPU 간 데이터 전송 최적화
        
    - Load manager
        - ~~Mini-batch 기반 계산 모듈과 task에 대한 데이터의 동적 GPU loading을 가속하는 memory loader 개발~~
        - ~~동시 요청되는 task 마다 GPU data loading을 비동기 처리로 병렬화~~
        - ~~Command Queue 기반 실행으로 GPU stall 최소화~~