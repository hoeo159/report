### 김동빈
+ Chunk Loader
	+ 전체 벡터 데이터셋이 저장된 storage에서 데이터를 읽어와 메인 메모리에 chunk 단위로 load
	+ 대용량의 데이터를 일정한 크기로 나누어, CPU가 접근 가능한 버퍼로 로딩
+ Disk Chunk Cache Manager
	+ 메인 메모리의 사용량이 한계에 도달했을 경우, 사용하지 않는 chunk를 제거하며 필요한 chunk만 유지
	+ Storage<->메인 메모리 간의 데이터 흐름을 캐시 구조로 관리
+ Chunk Scheduler
	+ 전체 chunk pair 간 거리 연산을 위한 스케줄링 로직 구성
	+ 연산 순서 결정, CUDA stream 분배, 중복 연산 제거 등 연산 계획 수립
+ Chunk Cache Manager
	+ GPU에서 연산을 수행하기 위해 필요한 chunk가 gpu 메모리에 존재하는지 확인하고 없을 경우 로딩
	+ 제한된 메모리를 효율적으로 활용하기 위해 LRU 방식 또는 “남은 연산 pair 수” 기준으로 캐시 교체 등을 통한 캐시 관리 수행
![[chunk-scheduler-cache-manager-sample.png]]

### 백민호
- Host-Device 간 task 별 query를 비동기적으로 수행하여 GPU의 stall을 최소화
- 대용량 dataset에 적합한 asynchronous model을 설계해야 함
	- CUDA event based synchronize
	- GPU HW scheduling in windows
- 아래 측정은 pinned memory에 모두 넣을 수 있는 작은 size에 대한 비동기 테스트  
before(kernel 내부에 loop 추가하여 시간 늘림, Nvidia Nsight System 분석)
![[overlap_not_async.png|350]]
after
![[overlap_async.png|350]]
- 현재 대용량 데이터에 대해서 data size에 비해 pinned memory가 부족하기 때문에 synchronize와 pinned memory에서 pageable memory로 memory transfer 과정이 병목이 됨
- memory scheduler
    - pinned memory를 최대한 크게 할당  
    - 각 stream은 scheduler를 통해 pinned memory의 가용 영역을 할당 받음  
    - stream 마다 각각 다른 task 부여  
    - pinned memory를 모두 다 쓰면 synchronize와 함께 pageable memory로 flush
- figure 예시
![[pinned_figure_sample.png]]
### 박태곤
+ gpu와 cpu의 빠른 데이터 통신을 위해서 cpu에 pinned memory할당 받아야됨

1. disk에 저장된 대용량 데이터를 mmap으로 cpu virtual address로 mapping
2. cpu와 gpu사이의 데이터 통신에 최적의 할당 가능한 pinned memory를 할당
3. 해당 pinned memory의 크기 만큼 disk의 데이터를 복사
4. 위의 과정을 파일의 모든 데이터를 읽을 때까지 반복 
5. 비어 있는 pinned memory가 있을 경우, 자동으로 disk에서 pinned memory로 데이터를 옮기는 작업 실행
6. load manager에서 하드웨어(gpu 메모리) 상황에 따라 할당할 수 있는 pinned memory크기를 정하고, 그 크기만큼 디스크에 저장된 데이터를 쪼개서 전달
7. 여러 개의 pinned memory를 할당 한다면, 복사를 완료해 다음 데이터를 처리해야 되는 영역을 탐색해 자동으로 disk에서 다음 데이터 load
pinned memory는 stream에서 복사되는 cpu 메모리 영역이다. 

#### Load Manager
* mmap() manager
	load manager는 mmap을 통해 virtual address 상으로 파일의 메모리 주소를 가지고 있음. 대용량 데이터 파일 명을 입력 받으면 mmap을 통해 virtual address를 매핑.
* pinned memory allocation manager
	pinned memory의 할당 상한은 cpu, gpu memory 가능 범위에 따라 다름. gpu에서 할당 가능한 global memory는 cuda api를 통해 얻을 수 있음. pinned memory의 양 만큼 gpu memory로 한번에 복사하므로, gpu memory size를 통해 얼마나 pinned memory를 할당할지 알 수 있음. 또한, gpu에서 연산을 위해 필요한 메모리 양을 체크하여, 유동적으로 할당, 해제 
* pinned memory load balancer 
	queue를 통해 디스크로부터 pinned memory로의 할당 명령을 쌓아둠. host to device 데이터 복사가 끝난 pinned memory가 있을 경우, queue로 새로운 데이터 복사 명령이 쌓이게 되고, 지속적으로 해당 pinned memory로의 데이터 복사가 이루어진다. 데이터는 디스크로부터 virtual address를 통해 가져오며, 여러 개의 블록으로 나누어서 블럭 단위로 가져옴.
