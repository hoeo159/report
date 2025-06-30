### 진행사항
#### Shared memory bank conflict 해결
* 기존의 chunk size로 나눠서 shared memory에 담고 실행하는 방식은 K(centroid의 개수)가 늘어날 때, shared memory에 모두 들어가지 않아 K를 또 다시 나눠야 하는 문제를 가짐
* Thread 한 개가 최대 1536차원의 dimension을 순차적으로 계산하며, 동시에 여러 thread가 같은 shared memory주소에 접근하기 때문에 bank conflict 발생
* dimension을 쪼개는 것이 아니라, 하나의 centroid의 전체 dimension을 모두 넣는 대신, K를 쪼개서 shared memory에 넣는 방식을 사용함.(최대 shared memory에 1536차원 기준 8개의 centroid가 들어감)
* 하나의 thread가 아닌 하나의 thread block이 데이터 포인트 한 개를 담당하도록 하여, 각각의 thread는 하나의 데이터 포인트의 각각의 dimension을 병렬적으로 처리하도록 함.
* 각각의 dimension은 shared memory의 순차적인 영역을 접근하기 때문에 bank conflict문제를 해결할 수 있음.
* 32개의 순차적인 thread가 순차적으로 데이터를 접근하도록 함.
### Plan
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