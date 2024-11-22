Christoph Peters, Tark Patel, Will Usher and Chris R. Johnson, 2023

Glyphs : 사람들이 a를 쓰는 방식이 다른 것처럼 같은 자형의 다른 모양 집합들
ODF(Orientation Distribution Function) : 3D 구 상에서 특정 방향으로 확산 확률을 나타내어 어떤 돌출된 모양을 나타내는 함수
[참고](https://www.shadertoy.com/view/dlGSDV)

### Introduction

&ensp;DWI라는 자기공명장치에 자주 쓰는 이미지가 있다. voxel마다 여러 벡터와 크기가 있어서 dimension이 높아 visualize하기 어려웠음. 그 중 HARDI(High angular resolution diffusion imaging)는 ODF를 측정하여 SH를 basis로 표현했다. 당연히 smooth할수록 많은 tessellation triangle이 필요하고 computation and bandwidth를 많이 먹는다. 이전 연구에서 ray casting은 glyphs수가 아니라 pixel 수를 따라가기 때문에 생각해볼만 했다. 당연히 ray marching step이 많을 수록 비용이 너무 많이 들었고 모든 ray가 camera에 발생, shadow를 위한 secondary ray가 없다고 가정했더니 당연히 빨라졌다.

&ensp;여기서는 SH glyph와 ray의 모든 intersection을 찾는 효율적인 방법(root finding) 소개한다. 또 SH basis로 만든 homogeneous formulation이 glyph surface normals에 더 쉬운 공식을 만들어준다. 이는 RT에 쓰는 AABBs에 대한 계산을 거의 정확하게 할 수 있다. 이전 work와 다른 점은 pre-computate 없이 임의의 ray에 대한 모든 intersection을 계산할 수 있게 해준다. 이를 통해 uncertainly visualization을 위한 volume rendering(잘 모름)과 soft shadow를 perform 할 수 있다.

### Related Work

### Ray-Glyph Intersection Test

#### Homogenizing Spherical Harmonics

&ensp;기존 $\theta$와 $\phi$를 쓰던 polar coordinate와 다르게 쓰기 쉽게 unit sphere위의 Cartesian coordinate를 사용한다. 대칭성을 위해서 짝수 level의 basis, 그 중 일반적으로 0, 2, 4 level만 사용한다, 또 홀수를 쓸 때에는 homogeneous를 할 때 $\sqrt{Y}$를 써야하는데 polynomial을 쓸 수 없다. 또 SH basis를 모두 동일한 차수의 homogeneous polynomial로 만들수 있다. 예를 들어 0, 2차항에 대해 각각 2, 0 degree의 basis를 곱해서 degree 4로 공통되게 뽑아낼 수 있다. k = 0, 2, 4에서 total basis는 15개이다.

#### Intersections as Polynomial Roots

&ensp;