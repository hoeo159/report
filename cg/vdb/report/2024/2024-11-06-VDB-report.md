#### 진행사항
* shared memory 사용한 성능 향상
* 자체점검실적 보고서 양식 수정

### Shared memory 사용한 성능 향상
* ClusterAssignment커널과 CentroidUpdate커널에서 shared memory를 사용하여 d_centroid와 d_clust_size의 중복 접근 줄임
* Centroidupdate커널에서 shared memory로 초기화한 후, 각 스레드가 atomicAdd 연산을 통해 centroid 좌표와 cluster 크기를 누적한다.
* 각 블록이 작업을 완료한 후 공유 메모리의 값을 전역 메모리로 업데이트하여 최종 결과 반영
#### K-means clustering 성능 측정 결과

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | ---------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 512k                      | 32                         | 128 | 1                         | 55.3646 -> 63.1644                                                     |
|          NVIDIA 4090           | 512k                      | 32                         | 128 | 1                         | 10.5712 ->                                                             |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | ---------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 128k                      | 2                          | 25  | 1000                      | 2487.77 -> 1651.42                                                     |
|          NVIDIA 4090           | 128k                      | 2                          | 25  | 1000                      | 291.152                                                                |

| GPU &#x20 &#x20&#x20&#x20&#x20 | Datapoint &#x20&#x20&#x20 | Dimenstion &#x20&#x20&#x20 | K   | iteration &#x20&#x20&#x20 | <span style="color: yellow;">execution time (ms)</span>&#x20&#x20&#x20 |
| :----------------------------: | ------------------------- | -------------------------- | --- | ------------------------- | ---------------------------------------------------------------------- |
|         NVIDIA 2080 TI         | 1M                        | 100                        | 25  | 1000                      | 45057.3 ->39092.3                                                      |
|          NVIDIA 4090           | 1M                        | 100                        | 25  | 1000                      | 10924 ->                                                               |

### OpenAI Embedding 사용
 * OpenAI 임베딩 API를 사용하여 텍스트 데이터를 1536차원 벡터로 변환
 * text-embedding-3-small 모델이 default로 텍스트를 1536차원으로 변환시켜주며 document(https://platform.openai.com/docs/guides/embeddings)의 use case인 Amazon fine-food reviews dataset을 사용하여 벡터로 변환

### Load manager 구현
* 