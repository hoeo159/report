1. abstract: 전체 완성 필요
	- what 한 문장
	- contribution에 대한 이야기 1-2문장
	- contribution 별 상세 내용 각 1-2문장씩
	- 결과에 대한 preview: 정확한 숫자를 넣지는 마시오.
	- 결과는 contribution과 반드시 연계하여 작성

시작 부분은 fourier series 기존에 어떻게 사용되고 있었는지
what : fourier series를 이용한 픽셀 연산으로 많은 양을 필요로 하는 effect들에 대해서 scalable하게 계산한다.
fourier series의 seperability를 이용하여 공통항들의 계수를 rendering 하기 전에 pre-compute 할 수 있다. veiling glare는 이러한 방법을 쓸 수 있는 대표적인 예시로 glare model의 복잡성에 상관없이 일정한 성능을 보장한다.
fourier series로 적용하기 위해 필요한 조건들 : periodic, integral에 대한 closed form
결과 : O(K)에 대해서 O(N * M)으로 변화 K는 glare model의 개수, N과 M은 2d fourier series에 대한 각 height, width에 대한 차수로 보통 같게 설정하기 때문에 $N^2$으로 봐도 된다. 결과에 대한 내용으로는 속도 뿐만 아니라 품질? 퀄리티 유지 정도에 대한 것도 체크해야 됨

5. Glare model
Veiling glare can be interpreted as a superposed response of weak and (potentially large) ghosts in optical system. Assuming each ghost $k$ is a 2D circle with constant intensity $I_{k}$, it cross-section through the center line can be represented as a simple gaussian function.

5.1 gaussian function

$$
f(x) \approx \frac{a_{0}}{2}+\sum_{n=1}^N a_{n}c_{n}(x)), \omega=\frac{2\pi}{T}
$$


fourier series는 컴퓨터 그래픽스에서 periodic signals의 표현, frequency-domain filtering 등의 목적으로 활용되고 있다. 하지만 그러한 특성 때문에 pixel 단위의 modeling 대한 직접적인 적용은 상대적으로 덜 연구되었다.
우리는 fourier series를 기반으로, 연산 비용이 높은 optical effects를 pixel 단위에서 scalable하게 rendering 할 수 있는 (framework? methodology?)를 제안한다. fourier basis의 seperability를 활용함으로써 rendering 전에 공통된 coefficient를 미리 precompute 할 수 있으며, 이는 effects의 scale에 무관한 constant한 성능을 보장한다.
이러한 방법의 대표적인 적용 대상은 veiling glare이다. 기존 방식은 많은 ghosts가 발생시키는 이미지를 누적하여 glare 효과를 구성했지만, 이를 2d fourier series를 이용한 parametric model로 대체하여 연산 비용을 크게 줄인다.
우리는 veiling glare가 fourier based modeling에 필요한 핵심 조건들; periodicity와 closed-form integrability을 만족함을 보이고 이에 기반한 precompute를 가능하게 한다.
(결과)Performance 측면에서 우리의 방법은 rendering cost를 기존 O(K)에서 O(N^2)으로 전환한다. K는 ghost 성분의 수, N은 fourier series의 order이며, 보통 width, height를 동일하게 설정하기 때문에 N^2으로 표현된다. 대부분의 경우에서 K는 N^2 보다 매우 크다(K >> N^2). 실험 결과, glare model의 complexity와 관계없이 scalable한 rendering performance를 보장하고, explicit한 방법과 유사한 quality를 제공한다.


우리는 complex effect primitives 를 위한 fourier series기반 efficient parametric computation을 제안한다.
기존의 effect 연산 방식과 다르게, parametric series computation은 연산 횟수가 effect의 개수에 의존하지 않는다.
해당 방식은 fourier series의 seperability 특성을 활용하여 rendering time에 series의 constant order만큼만 연산하여 실제 effect와 동일한 값을 연산한다.
이를 glare model에 적용하여 a number of partial glares의 누적에 대한 efficient한 parametric computation을 얻을 수 있었다. 

본 연구는 complex effect primitives에 대해 푸리에 급수를 기반으로 한 효율적인 parametric 계산 방법을 제안한다.  
제안하는 방식은 기존의 연산 방식과 달리, 연산 횟수가 effect의 개수에 의존하지 않는다. (일정하게 유지?)
푸리에 급수의 separability 특성을 활용하여 렌더링 시에는 고정된 차수의 급수 항만을 계산함으로써 실제 effect와 동일한 결과를 얻을 수 있다.  
이 방법을 glare 모델에 적용한 결과, 다수의 부분 glare 효과에 대한 누적 계산을 효율적인 parametric 방식으로 처리할 수 있음을 확인하였다.

본 연구는 multi-source light effects(복잡한 optical effects)에 대해 fourier series를 기반으로 한 효율적인 parametric 계산 방법을 제안한다(새로운 methodology를 제안한다).
fourier series의 separability 특성을 활용하여 rendering 이전에 공통된 fourier coefficient를 precompute 함으로써, effect에 대해 scalable한 performance를 유지할 수 있다.
대표적인 적용 사례로 veiling glare를 다루며, 기존의 ghost 기반 누적 방식 대신 2D fourier series를 활용한 parametric model로 대체한다.
특히 단일 glare model을 fourier based modeling에 적합한 수학적 조건 -periodicity, closed-form integration-을 만족하는 model로 표현이 가능함을 보이며, 이로 인해 효율적인 computation이 가능해진다.
우리의 접근은 glare model의 복잡도와 상관없이 constant한 성능을 유지하며, 실재와 유사한 high quality를 달성한다.

본 연구는 multi-source light effects(복잡한 optical effects)에 대해 fourier series를 기반으로 scalable하게 rendering하는 효율적인 parametric 계산 방법을 제안한다(새로운 methodology를 제안한다).
대표적인 적용 사례로 veiling glare를 다루며, 기존의 ghost 개수에 비례한 방식 대신 2D Fourier series를 활용한 parametric model로 대체한다. 
glare effect를 Fourier based modeling에 적합한 수학적 조건(periodicity, closed-form integration)을 만족함을 보이며, 이로 인해 precomputing이 가능해진다. 
실험 결과, 렌더링 비용은 O(K)에서 O(N²)로 개선되며(K ≫ N²), 품질 역시 기존 ghost 누적 방식과 유사한 수준을 유지한다.


2. intro: 전체 완성 필요 (4문단+contribution으로 작성)
	- 배경 1문단: 완전 일반적 배경은 이야기 하지 말고, 논문의 문제에 대해서 직접 바로 치고 들어가시오.
	- 기존 work의 문제점과 한계 1문단: related work의 3-4문장 summary라고 생각하고 작성
	- 우리의 approach에 대한 high-level 소개 1문단
	- 우리의 work에 대한 구체적인 문단 1문단: abstract의 확장 버전으로 생각해서 작성
	- contribution 2 or 3 가지 item

3. related work: 전체 완성 필요 (subsection 2-3개 정도 작성)
	- 위 배경 1문단에 쓴 내용을 확장해서 쓴다고 생각
	- Section 2 다음에 related work에 대한 intro 2문장 추가
	- subsection에 관련된 연구를 작성
	- 단순 논문 나열하는 것은 절대 금지
	- 각 문단별로 그 문단에 나오는 논문들에 대한 1문장 요약을 첫 문장으로 시작하기
	- subsection의 마지막은 우리의 일과 related work의 차이에 대해서 관계지어 기술 필요
