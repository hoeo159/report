### 제목 후보군 선정

- Fourier Series-Based Modeling for Scalable Glare Rendering
- Scalable Fourier-Based for Real-Time Rendering of Complex Effects
	- glare 제거 버전
- Parametric Rendering of Image-Space Optical Effects via Separable Fourier Series
	- gpt 추천
- Fourier-Based Parametric Rendering of Multi-Source Effects
- **Fourier-Based Parametric Rendering for Efficient Component Synthesis**
	- component 합성 기준으로(quad-based 참조)
- **Fourier-Based Rendering for Efficient Synthesis of Multi-Source Effects**
	- parametric 단어 제거 버전, component 대신 multi-source effect 추가
- Separable Fourier Series for Constant-Time Rendering of Accumulative Effects
	- separability와 constant time 강조

### abstract

본 연구는 multi-source로 구성된 effect를 efficiency하게 compute 할 수 있는 fourier series closed form solution을 이용한 계산법을 제안한다.

기존 방식은 수많은 components 개수에 dependent한 computation인 것과 비교해서 우리는 fourier series의 separability를 활용하여 rendering time에 constant order 연산과 동시에 high quality를 보장한다.

대표적인 예로, multi-source로 구성된 veiling glare 효과는 다수의 ghost image component의 누적에 의해 계산 비용이 증가하는 구조였으나, 이를 fourier series 기반의 단일 수식 모델(single-form expression)로 재구성함으로써, effect 개수에 독립적인 효율적 연산이 가능함을 확인하였다.  

또한 해당 모델이 fourier 기반 표현에 필요한 periodicity와 closed-form integration 조건을 만족함을 보이며, 다양한 accumulative 연산이 필요한 multi-source light effect에 적용 가능한 확장성 또한 확보하였다.

### introduction

intro: 전체 완성 필요 (4문단+contribution으로 작성)
	- 배경 1문단: 완전 일반적 배경은 이야기 하지 말고, 논문의 문제에 대해서 직접 바로 치고 들어가시오.
	- 기존 work의 문제점과 한계 1문단: related work의 3-4문장 summary라고 생각하고 작성
	- 우리의 approach에 대한 high-level 소개 1문단
	- 우리의 work에 대한 구체적인 문단 1문단: abstract의 확장 버전으로 생각해서 작성
	- contribution 2 or 3 가지 item
### 1차

lens 같은 optics based의 다양한 effects들은 multiple light source나 내부적인 reflection, diffraction 등으로 인해 생성된 수 많은 component들의 accumulate로 나타난다. 대표적으로 glare, bloom, flare와 같은 효과는 수 만개 이상의 광학적 성분이 중첩되어 형성되며 이미지의 특정 영역에 intensity가 spread되는 변화로 표현된다. 이러한 효과를 시각적으로 precise하게 재현하려면 각 component가 미치는 영향을 누적하여 계산해야 하지만, 이 과정은 많은 연산량을 필요로 하며 large scale에서 비효율적이다. 특히 multi-source light 환경에서는 component 수가 massive하게 증가하기 때문에, 성능 병목의 주요 원인이 된다. 이러한 문제를 해결하기 위해서는 효과의 복잡도와 무관하게 일정한 비용으로 계산 가능한 scalable한 계산 방법이 필요하다.

중첩된 primitives를 효율적으로 계산하기 위해서는, effect의 개별 성분의 중첩 연산 없이도 표현할 수 없는 수학적 표현이 필요하다. 이 때 fourier series는 complex function을 periodic basis function의 linear combination으로 표현함으로써, effect의 complexity나 scale에 무관한 constant한 성능을 보장한다. 특히 fourier series의 separability는 공간 상에서 각 axis에 대해 독립적으로 처리할 수 있도록 하여 high dimension computation도 구조적으로 단순화 할 수 있는 장점이 있다. 이러한 특성을 이용해 수 많은 primitive 효과 전체를 fourier coefficient의 order 수준으로 표현하는 것을 가능하게 만든다.

fourier series나 transfer를 이용한 다른 논문들 소개~ 선행 연구들은 실용적으로 유의미한 도구다
(조사 후 추가 예정)

fourier series를 rendering에 직접 적용하기 위해서는, 해당 함수의 몇 가지 수학적 조건을 만족해야 한다. 가장 핵심적인 조건은 periodicity으로, fourier series는 본질적으로 주기를 갖는 basis 함수의 weighted summation이기 때문에 대상 효과가 일정 범위 내에서 반복되거나, 유사한 주기적 구조를 가질 때 근사가 효과적으로 작동한다. 두 번째 조건은 (closed-form integration)이다. 이는 Fourier 계수를 계산하기 위한 적분을 numerical하게 반복 계산하지 않고, 사전에 명확한 수식으로 표현할 수 있어야 함을 의미하며, 효율적 렌더링을 위한 선행 조건이 된다. 이러한 조건들은 일반적인 이미지 기반 효과에 바로 적용하기 어려울 수 있으나, 특정 클래스의 효과들 - 예를 들어, spatially smooth하고 구조적으로 반복되는 glare field -에는 적용 가능성이 충분하다. 본 연구에서는 이러한 수학적 조건을 만족하는 glare model을 구성하고, 이를 통해 fourier 기반 표현이 실제 렌더링에 적용 가능함을 보인다.

(veiling glare 소개는 기존 introduction 참조)

Veiling glare와 lens flare는 lens barrel 내부 및 이미지 센서 표면 간의 반사로 인해 발생하는 stray light(ghost)의 누적 효과로 알려져 있다 \[Naylor 1970]. 일반적으로 lens flare는 상대적으로 강한 강도, 날카로운 윤곽, 작은 면적으로 구성되는 반면, veiling glare는 더 넓게 퍼지고 경계가 부드러우며, 시각적으로 흐릿한 인상을 남긴다. 이러한 효과들은 과거에는 촬영 과정에서 제거해야 할 artifacts로 간주되었지만, 최근에는 사실감과 몰입감을 높이는 시각적 요소로 재조명되며 의도적으로 표현되기도 한다. 특히 veiling glare는 고휘도 광원 주변에서 나타나는 낮은 주파수의 intensity spread 현상으로, 다양한 실내외 광 환경에서 중요한 시각적 단서로 작용한다.

기존의 veiling glare 렌더링 기법은 수십 개에서 수백 개에 달하는 ghost path를 추적하는 경로 기반 광선 추적(path tracing)에 기반한 방법이 대표적이다. 이 방식은 높은 물리적 정확도를 제공하지만, 하나의 고품질 이미지를 생성하는 데 수 시간이 소요되는 등 실시간 응용에는 적합하지 않다. 이를 개선하기 위해 Hullin et al.은 ray tracing과 rasterization을 결합한 하이브리드 방식을 제안하였고, 이후에는 행렬 기반의 flare mapping 기법이 소개되며 실시간 렌더링 가능성도 열렸다. 그러나 이러한 방식들 역시 ghost의 수에 따라 성능이 선형적으로 저하되는 한계를 지니며, veiling glare처럼 ghost 수가 많고 low-frequency로 확산되는 효과에는 최적화되어 있지 않다. 최근에는 HDR 영상 처리 문맥에서 glare component들을 flare와 veiling glare로 분리하여 각각 rendering하는 방식 \[Rouf et al. 2011]이 성능 개선 측면에서 유리함을 보였다. 본 연구는 veiling glare에 대해 parametric 모델을 적용함으로써, 기존 ghost 누적 방식 대비 수십 배 빠른 렌더링 성능과 시각적으로 유사한 품질을 동시에 달성한다.

본 연구는 veiling glare를 포함한 multi-source 기반 optical effect를 효율적으로 근사하기 위해 fourier series 기반의 parametric rendering 기법을 제안하며, 다음과 같은 주요 기여를 포함한다:

- 1 추가 예정
- 2 추가 예정

### 2차

lens 기반의 glare, flare, bloom 등 다양한 effects는 multiple light source나 내부적인 reflection, diffraction 등으로 인해 생성된 수 많은 component들의 accumulate로 나타난다. 이러한 effect를 high quality로 rendering하기 위해서는 각 성분을 개별적으로 계산하고 누적하는 접근을 취해 왔지만, 이러한 방식은 효과의 복잡도에 따라 연산량이 선형 이상으로 증가하는 구조적 한계를 지닌다. 우리는 이러한 누적 기반 접근의 한계를 극복하고, effect의 complexity에 관계없이 constant한 계산 구조를 갖는 새로운 표현 방식(parametric representation)을 제안한다. 이 방식은 effect의 구조를 수학적으로 modeling함으로써, components에 scalable하게 multi-source effect를 계산할 수 있도록 한다.

본 연구의 핵심 접근은 fourier series based computation이다. complex function을 periodic basis function의 linear combination으로 표현함으로써, effect의 complexity나 scale에 무관한 constant한 성능을 보장한다. 특히 fourier series의 separability는 공간 상에서 각 axis에 대해 독립적으로 처리할 수 있도록 하여 high dimension computation도 구조적으로 단순화 할 수 있는 장점이 있다. 이러한 특성을 이용해 수 많은 primitive 효과 전체를 fourier coefficient의 order 수준으로 표현하는 것을 가능하게 만든다.

fourier series를 rendering에 직접 적용하기 위해서는, 해당 함수가 몇 가지 수학적 조건을 만족해야 한다. 우선 표현 대상의 periodicity으로, fourier series는 본질적으로 주기를 갖는 basis 함수의 weighted summation이기 때문에 대상 효과가 일정 범위 내에서 반복되거나, 유사한 주기적 구조를 가질 때 근사가 효과적으로 작동한다. 두 번째 조건은 (closed-form integration)이다. 이는 Fourier 계수를 계산하기 위한 적분을 numerical하게 반복 계산하지 않고, 사전에 명확한 수식으로 표현할 수 있어야 함을 의미하며, 효율적 rendering을 위한 선행 조건이 된다. 이러한 조건들은 일반적인 이미지 기반 효과에 바로 적용하기 어려울 수 있으나, 특정 클래스의 효과들 - 예를 들어, spatially smooth하고 구조적으로 반복되는 glare field -에는 적용 가능성이 충분하다. 본 연구에서는 이러한 수학적 조건을 만족하는 glare model을 구성하고, 이를 통해 fourier 기반 표현이 실제 렌더링에 적용 가능함을 보인다.

이러한 수학적 조건은 veiling glare와 같이 spatially smooth하고 low-frequency 특성을 지닌 효과에 잘 부합하며, 본 연구에서는 이를 대표 사례로 삼아 제안한 방법론의 효과를 검증한다. 기존의 veiling glare 렌더링 기법은 수십 개에서 수백 개에 달하는 ghost path를 추적하고, 이들을 화면 상에 누적하여 결과를 합성하는 방식이 일반적이다. 이와 같은 방법은 광학적으로 높은 정밀도를 제공하지만, 구성 요소 수에 따라 연산량이 선형 이상으로 증가하며 실시간 rendering에는 적합하지 않다. 본 연구에서는 이러한 누적 기반 접근을 대체하기 위해, veiling glare를 2d fourier series로 구성된 analytic model로 표현하는 계산 방식을 제안한다. 이 방식은 rendering 전에 fourier coefficient를 사전 계산하고, 이후에는 basis function의 linear combination만을 compute함으로써 ghost 수에 관계없이 constant한 performance로 glare effect를 표현할 수 있도록 한다. 

본 연구는 veiling glare를 포함한 multi-source effects를 효율적으로 근사하기 위해 fourier series based (parametric) rendering 기법을 제안하며, 다음과 같은 주요 contribution을 포함한다:
- 1 추가 예정
- 2 추가 예정
### teaser figure
![[0520_teasers.png]]