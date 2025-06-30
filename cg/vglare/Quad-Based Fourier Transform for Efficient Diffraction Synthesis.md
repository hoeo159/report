
Far-field diffraction은 이미지 공간에서 Discrete Fourier Transform (DFT)을 사용하여 평가할 수 있지만, 조밀한 샘플링으로 인해 비용이 많이 듭니다.

우리는 간단한 벡터 primitives (quads)에 대한 continuous Fourier transform의 closed-form 솔루션에 기반한 기술을 제안하고 실시간 성능을 달성하기 위해 계층적이고 점진적인 평가를 제안합니다.

우리 방법은 광학 시스템에서 회절 효과를 시뮬레이션할 수 있으며 동적 광원으로 인한 다양한 가시성을 처리할 수 있습니다.

또한, near-field diffraction으로 원활하게 확장됩니다.

우리는 현실적인 실시간 glare 및 bloom 렌더링을 포함하여 다양한 애플리케이션에서 우리 솔루션의 이점을 보여줍니다.

vs

본 연구는 multi-source로 구성된 effect 연산을 효율적으로 처리할 수 있는 새로운 fourier series 기반의 rendering method를 제안한다.

기존 방식은 수많은 components 개수에 dependent한 computation인 것과 비교해서 우리는 fourier series의 separability를 활용하여 rendering time에 constant order 연산과 동시에 high quality를 보장한다.

대표적인 예로, multi-source로 구성된 veiling glare 효과는 다수의 ghost image component의 누적에 의해 계산 비용이 증가하는 구조였으나, 이를 fourier series 기반의 단일 수식 모델(single-form expression)로 재구성함으로써, effect 개수에 독립적인 효율적 연산이 가능함을 확인하였다.  

또한 해당 모델이 fourier 기반 표현에 필요한 periodicity와 closed-form integration 조건을 만족함을 보입니다.

우리는 accumulative 연산이 필요한 다양한 multi-source effect에 우리 솔루션의 이점을 보여줍니다.
