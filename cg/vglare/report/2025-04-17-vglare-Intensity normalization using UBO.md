
### 이번 주 진행사항
-  UBO를 이용한 min-max mipmap normalization
### UBO min-max mipmap normalization 학습
- UBO를 통해 last_mip 정보를 shader에 전달하고 이를 활용하여 normalize를 수행
- last_mip을 저장하는 uniform buffer UMX 생성
- UMX buffer를 SSBO로 binding하기 위해 `bind_as` 사용
	- `UMX->bind_base_as(GL_SHADER_STORAGE_BUFFER, 7)`
	- `glBindBufferBase`의 wrapper, `bind_as`와 `bind_base` 합친 버전
	- `effect->bind_shader_storage_buffer`에서도 동일한 기능을 수행
		- `bind_base_as`에 비해서 variable name으로 `get_shader_storage_block_binding` 과정이 추가
	- 지정된 target(GL_SHADER_STORAGE_BUFFER)의 index(7) 위치에 buffer(UMX) binding하여 shader가 해당 index 타고 buffer access
	- SSBO처럼 read&write 가능

### UBO min-max mipmap normalization rendering process
- compute glare process
	- \[compute_glare] DST 텍스처에 pixel별 각 ghost의 2d gaussian intensity compute
- minmax process
	- \[build_minmax] MMX 텍스처에 DST 텍스처 정보를 사용하여 minmax mipmap 생성
	- \[copy_last_mip] UMX 버퍼 객체를 `bind_base_as()`로 SSBO binding 후 
	  MMX->last_mip 정보를 가져와서 UMX 버퍼 객체에 vec2 type 저장
	- \[normalize] UMX 버퍼 객체에 저장된 DST 텍스처의 min, max 정보를 사용하여 normalize 진행

<div style="page-break-before: always;"></div>

### benchmark 결과
- texture (sampler2D) based normalize
	- mean GL time  = 0.362 ms
	- VGlare_FSD2 = **0.257** ms
- UBO based normalize
	- mean GL time  = 0.376 ms
	- VGlare_FSD2 = **0.261** ms
- unnormalize (minmax process 포함)
	- mean GL time  = 0.359 ms
	- VGlare_FSD2 = **0.257** ms
- unnormalize (minmax process X, normalize 이전 버전)
	- mean GL time  = 0.273 ms
	- VGlare_FSD2 = **0.177** ms

