### 진행사항

#### Shared Memory 및 Chunk 추가
+ GPU의 shared memory는 용량이 제한적-> 모든 데이터를 한번에 shared memory로 로드하는 것은 불가능
+ 기존의 shared memory 구현 방식은 shared memory의 용량이 한계에 다다른 경우 K-means clustering이 불가능
+ 작은 데이터셋에서는 효과적으로 clustering하지만 제안서의 1536차원 100만개를 random dataset으로 실험해본 결과 memory 부족
+ 기존의 shared memory를 사용하는 방식에서 데이터를 chunk로 분할하는 작업 진행
+ Code
```
__device__ int d_chunk_size = 256;
```
	- chunk size 선언
	
```
__device__ float partialDistance(const float* point, const float* partial_centroid, int chunk_size, int offset) { 
	float dist = 0.0f; 
	for (int i = 0; i < chunk_size; ++i) { 
			float diff = point[offset + i] - partial_centroid[i]; dist += diff * diff; 
	} 
	return dist; 
}
```
	- 차원을 청크 단위로 분할하여 거리 계산


+  chunk size 별 k-means clustering 성능 측정

| Chunk Size | Execution Time (ms) |
| ---------- | ------------------- |
| 512        | x                   |
| 384        | 431352              |
| 256        | 322852              |
| 128        | 208178              |
| 64         | 197330              |
| 32         | 210979              |
| 16         | 141477              |
| 8          | 139204              |
| 4          | 140833              |
| 2          | 223940              |




#### OpenAI 임베딩 데이터셋
+ 제안서 기준 : openai_large_5m dataset
	+ nytimes보다 차원수가 6배 높고, 데이터셋 크기가 3배
	+ 데이터셋은 제안서 기준 100만개
	+ https://github.com/pgvectorBench/pgvectorBench -> 1536차원 500만개 데이터셋이 50만개씩 10개로 분할되어 있음
+ 구매 필요 없음 -> 데이터셋 다운받아본 결과, 이미 1536차원으로 embedding 완료
+ 데이터셋은 parquet 형식으로, cpp에서 사용가능하도록 변환 필요

### Plan
+ 연차보고서 및 차년도 계획서 작성
	+ 차주내로 작성 필요한 부분 안내 받고, 12월 초까지 작성해야함
+ parquet 형식 데이터 로드 코드 작성
+ chunk 코드 다시 작성
+ 최적화 - memory align 코드 작성