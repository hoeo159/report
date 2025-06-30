## 김동빈
### GPU 기반 벡터 간 거리 계산 가속 기술 개발
#### GigaVoxel 기반 memory load manager
+ GigaVoxel의 구조를 참고하여 memory load manager 설계 아이디어 고안
	+ Block Loader, Block Cache Manager, Block Scheduler, Distance Kernel로 이루어짐
		+ Block Loader
			+ GigaVoxel에서 brick 단위로 분할하여 필요한 부분만 GPU에 로딩하는 것처럼 거리 계산에 필요한 vector block만 로딩
		+ Block Cache Manager
			+ GPU 메모리에 올라간 block들의 상태를 추적하고, 한도를 초과할 경우 가장 오래된 block을 제거
		+ Block Scheduler
			+ 전체 block 간 (i, j) 쌍에 대해서 pairwise 거리 계산 수행 -> 중복 계산 방지를 위해 (i<=j) 형태로 Scheduling
			+ GigaVoxel에서 렌더링과 데이터 로딩이 동시에 진행되는 것처럼 데이터 로딩과 연산을 분리하여 stall 최소화
		+ Distance Kernel
			+ 병렬로 거리 계산 진행
+ todo
	+ gigavoxel 문서 보고 좀 더 구체적인 구조 확인
	+ Out-of-core data 환경 base 코드 작성
		+ 임의의 한도 설정 후 management 진행하는 방식
#### Distance query
+ 벡터 간 거리 계산 가속 실험을 위한 base 코드 개발
	+ random vector dataset generation
		+ K개의 중심을 기반으로 N개의 고차원 벡터 생성
	+ Query vector generation
		+ 특정 cluster 중심 주변에 위치하도록 query vector 생성
		+ 랜덤으로도 생성 가능
	+ CUDA 커널을 이용한 거리 계산
		+ Query vector가 주어지면, CUDA 커널을 통해 벡터DB 내 모든 벡터와의 거리를 병렬로 계산
		+ 가장 유사한(가장 가까운) 벡터를 찾아 인덱스, 거리, label 등 결과를 출력
		+ 거리 계산에 걸린 실행 시간 및 Top 10 nearest vectors 출력
	+ CPU 연산과 비교
		+ 동일한 query, DB에 대해 CPU로 거리 계산 수행
		+ 마찬가지로 거리 계산에 걸린 실행 시간과 Top 10 nearest vectors 출력
+ 코드 실행 결과 - GPU, CPU 연산 실행 시간 비교
	+ 상황
		+ 512차원의 100,000개 vector data
		+  query vecotr 하나가 주어지면 위 100,000개 vector data 모두와 거리 계산 진행
	+ L1 distance
		+ 0.9728 ms
	+ L2 distance
		+ 0.997376 ms
	+ Cosine similarity
		+ 1.06598 ms
	+ Cosine similarity가 추가적인 연산이 존재하여 좀 더 많은 시간이 걸림 -> 계산 전 정규화를 통해 dot 연산만으로 바로 할 수 있도록 한다면 시간이 줄어들 것으로 예상됨
+ todo
	+ 2가지 상황에 대한 base 코드 작성 -> 검색(기존)+빌드(추가)
		+ 현재 base 코드는 query가 주어지면 db 내의 데이터들과의 거리를 구하는 검색 상황에 대한 코드인데, 이에 더해서 db 내에서 vector들 사이의 거리를 구하는 빌드 상황에 대한 코드 작성
	+ Cosine similarity 계산 단순화를 위한 normalize 진행

## 백민호

#### CUDA asynchronized를 위한  Stream Load Balancer 설계

- kernel과 data transfer의 asynchronous launch
	- GPU loading 지연 문제를 완화하기 위해서 각 함수의 비동기 실행 구조가 필요
	- 추가적으로 task 별 query가 주어졌을 때, command queue를 이용하여 비동기적으로 수행
	- 현재 방법으로는 kernel 간 overlap만 확인할 수 있고 data transfer 과정에서 다른 stream의 kernel 비동기 실행 구현 필요
		- 현재 asynchronous 코드는 약 15%의 성능 개선
	- CUDA event based synchronize
		- stream 간 동기화 시점을 좀 더 타이트하게 설정 할 수 있음
- 고차원 벡터 데이터 셋에 특화된 비동기 task queue
	- 현재 단순 CUDA asynchronous code는 거의 serial 하게 실행되어있기 때문에 소프트웨어 수준에서 개선이 필요
	- cuda work queue 상단의 layer
		- stream 환경에서 실행될 작업들을 dynamic batch size로 쪼개고, 적절한 stream에게 비동기적으로 분배하는 task queue
	- GPU stream 상태 기반의 stream handler(stream priority, preemption 등)
		- CUDA stream의 idle 여부를 확인하기 위해 stream 실행 상태를 pooling 하거나 event를 이용하여 대기
		- stream 간 dependency와 동기화 시점 관리
	- task queue는 priority based scheduling 또는 dependency 기반 graph(DAG) 구조 확장 가능
- todo
	- event 기반 synchronize stream execution 구현 및 성능 개선 체크
		- 현재 loop 내부에 동기화 진행 시 병목이 생기며, loop 밖에서는 large dataset에 대한 segment fault 에러 발생
	- stream dependency와 priority 등 status 체크 handler 기본 설계

## 박태곤

#### Host-Device Data Transfer Strategy
데이터 전송과 관련하여 사용자가 명시적으로 host device memory copy를 통해 전달할 수도 있지만, CUDA에서 제공하는 unified memory와 zero-copy를 이용하여 더 쉽게 접근할 수 있음.
* Unified Memory
	CPU, GPU에서 동일한 주소 접근을 통해 일반적인 메모리 접근을 가능하게 하는 통합 메모리 영역.
	* Unified virtual addressing 기능을 지원하여, host memory와 gpu memory가 virtual하게 통합되어, host에서 사용한 동일한 pointer로 device에서 사용할 수 있음.   
* Zero-Copy
	* Host의 메모리를 Device에서 동일한 물리 메모리로 접근하여, 데이터 복사 없이 접근할 수 있음
	* Host와 Device 각각에 동일한 데이터를 복사할 필요가 없으므로, 전체 시스템 메모리 사용량을 줄일 수 있음. 또한, Device 메모리의 크기를 고려하지 않고, host memory까지 사용 가능.
	* Device에서 지속적으로 대용량 데이터에 접근하여 연산할 경우, PCIe bus의 대역폭 한계로 인해 data copy 후 사용하는 것이 더 효율적일 것으로 생각됨.
#### Optimizing Data Transfer in CUDA C/C++
https://developer.nvidia.com/blog/how-optimize-data-transfers-cuda-cc/

* Measuring Data Transfer Times
	데이터 전송 시간은 nvprof 을 이용한 CUDA events를 활용하여 측정할 수 있음.
```
$ nvcc profile.cu -o profile_test
$ nvprof ./profile_test
```
nvprof를 활용하여 각각의 CUDA memcpy 콜에 의한 시간을 측정할 수 있다. 추후 data transfer 최적화를 진행할 경우 사용할 수 있음.
* Pinned Memory
	일반적인 메모리를 할당하여 host와 device의 통신에 사용하면, pageable memory에서 pinned memory로의 transfer이 필요하지만, pinned memory로 바로 할당하고 사용하면, 그 cost를 줄일 수 있음.

### Load Manager 구현 계획
가변적인 데이터 용량에 대한 처리를 위해 전체 데이터를 small batch로 쪼개서 data transfer 후 kernel 작업
#### Data Batch Size
* warp 단위의 배수로 data batch size 설정 계획
	shared memory에 데이터를 load시, 32개의 bank 단위로 데이터가 나눠져 들어가기 때문에(Shared memory has 32 banks that are organized such that successive 32-bit words map to successive banks. Each bank has a bandwidth of 32 bits per clock cycle., https://docs.nvidia.com/cuda/cuda-c-programming-guide/) bank conflict를 고려할 때, 32 * 4 bytes의 배수 단위로 batch size로 설정.
#### 대용량 dataset read
* mmap 
	대용량 data point는 disk에 저장될 것이며, 이를 host에서 자유롭게 읽기 위해 mmap을 활용하고자 함. mmap은 파일을 프로세스의 virtual memory에 mapping하여, File I/O 호출 없이 메모리 읽는 것처럼 파일의 데이터를 읽을 수 있도록 함.
* batch size segmentation
	주어진 명령어의 데이터 범위에 따라 파일로부터 데이터를 읽음. 이 때, 128bytes 배수의 batch size만큼 데이터를 쪼개서 sequential하게 device memory로 복사.
* 연산 후 결과 저장할 메모리 할당
	연산 후, 연산 결과를 저장해야 될 경우, device에서 host로 결과를 가져와야 하기 때문에 pinned memory로 할당. 이 때, device에서 host로 데이터 전송이 sparse하게 일어날 것으로 예상되기 때문에 데이터 복사를 하지 않는 zero-copy 활용을 고려.