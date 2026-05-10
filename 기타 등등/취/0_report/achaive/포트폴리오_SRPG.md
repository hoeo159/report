### Generative AI 기반 SRPG (2025.09 ~ 2026.05)

#### 프로젝트 개요

&ensp;포스트 아포칼립스 세계관의 SRPG에서 플레이어의 행동과 대화 기록이 NPC의 태도, 이벤트 분기, 세계 상태에 누적으로 영향을 미치는 동적 인터랙티브 게임 시스템을 개발하였습니다. Unity 6.3 기반으로 거점(Hub) - 탐험(Expedition) 루프를 설계하고, LLM과 규칙 기반 AI를 결합하여 매 플레이마다 다른 서사가 만들어지는 구조를 구현했습니다. ([링크](https://github.com/hoeo159/SKKU_SRPG_AI))

#### 주요 역할

##### 1. LLM 통합 및 프롬프트 엔지니어링

- OpenAI Responses API와 Unity `UnityWebRequest`를 통해 비동기 호출 클라이언트 구현
- 5종 세계 파라미터 + 8종 플레이어 성향 + NPC 호감도/메모리를 자연어 라벨로 변환하는 `PromptContextBuilder` 작성 (변수명·수치 노출 없이 정성적 표현으로 매핑)
- LLM 응답을 `reply / affinityDelta / memorySummary / tags` JSON 스키마로 파싱하고, 호감도 변화량 ±30 Clamp 및 응답 180자 제한으로 안정성 확보
- 사용자 입력을 별도 `PLAYER_SAYS` 블록으로 격리하여 프롬프트 인젝션 방어
- 탐험 종료 시 보고서 UI와 함께 다음 Hub 이벤트 LLM 생성을 백그라운드 Coroutine으로 처리하여, Hub 진입 시 캐싱된 결과를 즉시 표시

##### 2. Utility AI 기반 전투 시스템

- 12×12 격자 SRPG 환경에서 NPC가 다중 요소 점수 합으로 최적 행동을 선택하는 Utility AI 구현
- 공격력, 처치 가능성, 거리와 같은 5가지 요소와 동점 방지 노이즈, 위험 회피 등의 추가 로직을 통해 점수 계산
- BFS 경로 탐색으로 이동 가능 타일을 산출하고, 각 후보 위치마다 사정거리 내 모든 타겟을 평가

##### 3. 시스템 설계 및 모듈 통합

- `ScriptableObject` 기반 데이터 구조(`GameStateSO`, `EventCardSO`, `UnitDataSO`)로 게임 상태와 콘텐츠를 분리, 디자이너가 코드 수정 없이 이벤트·NPC를 추가할 수 있도록 설계
- 탐험 행동 통계(채집·대화·전투·회피)를 8종 성향 델타로 변환하는 `ProfileCalculator` 구현 
- 이벤트 선택은 성향 가중치 합산(`baseWeight + Σ(wTrait × normalizedTrait)`)에 이벤트 저장 큐를 결합하여 반복을 방지하면서도 플레이어 성향에 맞는 이벤트가 등장
- 닫힌 루프 설계 : 탐험 행동 → 성향 변화 → 이벤트 선택 가중치 → 이벤트 결과 → 세계 파라미터 변화로 이어지는 피드백 구조 구축

<div style="page-break-before: always;"></div>

#### 결과

- LLM 호출 결과를 이벤트 ID 단위로 캐싱하여 재진입 시 API 비용 절감 및 응답 일관성 확보
- 동일한 시나리오에서 플레이어 성향에 따라 서로 다른 이벤트, NPC 반응이 발생하는 동적 분기 시스템 완성
- 게임 기획, LLM 연동, 전투 AI, UI까지 개발자 관점에서 풀스택 게임 시스템을 설계, 구현 경험

<div style="text-align: center;">

  

![[탐험 진행(이동, 공격).png]]

  

<small>- 탐험 진행(이동, 공격) -</small>

  

</div>


<div style="text-align: center;">

  

![[대화와 이벤트.png|450]]

  

<small>- NPC 대화와 이벤트 생성 -</small>

  

</div>
