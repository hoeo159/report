### 논문 진행 사항
#### Labeled dataset 생성
+ clustering이 잘 되었는지 직관적으로 확인할 수 있도록 labeled dataset 생성 코드로 수정
+ 또한, 기존의 데이터 생성 코드는 데이터 포인트들이 너무 겹쳐져 있어서 제대로 된 clustering이 불가능 -> 각 데이터 군집 별로 거리가 조금 더 있도록 코드 변경
+ K개의 중심 좌표 무작위 생성 후 N개의 데이터 생성 및 label 부여
+ Standard K-means 결과
  ![[labeled-dataset-standard-kmeans.png|400]]
#### Hierarchical K-means
1. 초기 grouping -> LSH grouping으로 바꿀 예정 
	+ 전체 데이터를 대상으로, K-means clustering을 통해 g_init개의 초기 그룹들 생성
2. 각 그룹의 중심점을 가지고 K-means clustering 수행
	+ 이 때, 각 그룹의 크기(속해있는 데이터 개수)를 가중치로 주어 해당 clustering의 중심점 업데이트
	+ e.g., 그룹 A (50개 데이터), 그룹 B (10개 데이터), 그룹 C (1개 데이터)일 때, 
		+ 새로운 cluster 중심점 = (그룹 A 중심점 x 50 + 그룹 B 중심점 x 10 + 그룹 C 중심점 x 1) / 61
3. 각 그룹을 여러 하위 그룹으로 분할 (압축 해제) 
	+ 해당 그룹 내에서 K-means 수행하여 s개의 그룹으로 분할
4. 다시 전체 그룹에 대해 K-means clustering 수행
5. 3, 4 과정을 reps만큼 반복
6. 각 데이터 포인트에 대해 자신이 속한 최종 그룹의 cluster id를 할당

#### Todo
+ 하위 그룹 분할 방법 K-means 말고 다른 방법 찾기
+ 초기 그룹을 몇 개로 설정할 것인지(현재는 최종 클러스터 갯수 x 5), 압축 해제를 몇 개로 할 것인지에 대해 고민

#### Hierarchical K-means 실험 결과
* Dataset
	* test위한 N=10000, K=3, split num=5, dimension=100
	* 기본 K-means
		* Standard K-means time = 11353 ms, tandard K-means SSE = 1.001176374
	* Hierarchical K-means
		* level=1
			* ![[level1.png|400]]
		* level=2
			* ![[levle2.png|400]]
		* level=3
			* ![[level3.png|400]]
		* level=10
			* ![[level10.png|400]]

#### 실험 시각화 및 Plan
* level에 따른 K-means clustering SSE 및 execution time 측정
* ![[level_vis.png]]
* split num에 따라 측정 예정

