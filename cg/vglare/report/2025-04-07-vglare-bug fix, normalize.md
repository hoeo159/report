### 이번 주 진행 사항

- 코드 수정 과정에서 오류 수정
	- x, y 축으로 intensity가 늘어지는 현상(우측 사진이 원본)
![[0407_minor_bug.png|200]]     ![[0407_minor_.png|200]]
- intensity를 clamp로 자르는 것에서 normalize로 수정

### intensity 번짐 수정

- coefficient를 구하는 $\kappa$를 구하는 과정에서 약간 수정사항이 있었음
	- `n || m` 에서 `!n || !m`으로 수정
![[0407_minor_bug_fixed.png|200]]     ![[0407_minor_fixed.png|200]]

### intensity normalize

- mipmap texture 생성 후, last_mip의 min, max를 통해 normalize
- 결과(K = 100, N, M = 15, sigma = 0.7, T = 20.0)
  좌측 사진 normalize, 우측 사진 clamp
![[0408_after_nor.png|200x150]]     ![[0408_before_nor.png|200x150]]