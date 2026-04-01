---
name: Kong82
tools: [Unity, C#, ECS]
image: assets/kong.svg
description: Kong Studios 재직 중 작업한 프로젝트
---

# 1. 프로젝트 개요

- RTS / Mobile (Android, iOS)
- 클라이언트 프로그래머 (2025.07 ~ 2026.03)
- 소형 팀 (20인 미만)
- 결정론적 락스텝 기반, FixedPoint 사용, 대규모 시뮬레이션에 ECS 활용

# 2. 담당 업무 개요

- DOTS + ECS 구조로 리팩토링
- 저사양 최적화 (ARM Profiler, Unity Profiler, RenderDoc 활용)
- FX 툴 및 FX 바이너리 데이터 구조 제작
- 신규 모드 (배틀로얄, 넥뿌) 프로토타이핑
- 1vs1만 지원하던 구조에서 다인 멀티플레이, 팀전으로 스펙 업그레이드

# 3. 업무 상세

## 1. DOTS + ECS 리팩토링

#### 작업 배경

- 10인 배틀로얄 모드가 추가되면서 한 클라이언트 내 처리해야 하는 유닛의 규모가 대폭 상승함 (기존 80 → 변경 후 200)
- 아트스타일 리워크로 인해 타겟 모바일 기기에 대한 부하가 심해짐
- 프로젝트 프로토타입이 MonoBehaviour를 과도하게 많이 사용하고 있어 코드 흐름을 파악하기 어렵고, 이로 인해 디싱크가 발생하였음
- 기획팀에서 프로토타입에 구현된 유닛들의 움직임 (특히 길찾기, 로컬 회피)에 대해 플레이 경험이 좋지 않다고 피드백

#### ECS 도입 사유

- 메모리에 엔티티가 archetype/chunk 단위로 순차적 저장되기 때문에 캐시 친화적, 대량 시뮬레이션에 유리
- Job System을 활용하여 데이터 참조→상태 변경의 흐름을 자연스럽게, 그리고 병렬적으로 처리될 수 있도록 구현하면서 Burst를 통한 추가 최적화로 성능 향상
    - 모바일 기기에서 FixedPoint 연산의 비용, 부가적으로 발생하는 CPU 코스트를 멀티쓰레딩으로 우회하려는 목적 포함
- 코드의 흐름을 일직선 시스템으로 단순화하여 빠른 디싱크 원인 파악 및 해결
- 컴포넌트/시스템으로 분리된 구조를 통해 명확한 목적 표시, 코드 가독성 향상
- 게임 로직과 비주얼 오브젝트를 완전 분리하여 신규 유닛 구현, 유닛 스킨 적용 플로우 개선

#### 작업 내역

##### 1. ECS 시스템 구조 정의

- ECS에서는 각 시스템을 업데이트 그룹으로 묶어서 관리 가능, 그룹의 서브그룹도 정의 가능
- SimulationBootstrapGroup, SimulationTickGroup 및 하위 그룹의 시스템들은 기존 시스템 리스트에서 제외되고 별개의 시뮬레이션 러너가 직접 Update 호출
    - SimulationBootstrapGroup: 유닛 소환, 버퍼 기록용 월드 엔티티 소환 (one-shot)을 담당할 시스템들
    - SimulationTickGroup: 시뮬레이션 틱 위에서 도는 시스템들의 상위 그룹
        - BeginSimulationTickEntityCommandBufferSystem/EndSimulationTickEntityCommandBufferSystem: 틱 그룹용 ECB
        - SimulationStatsGroup: 스탯 업데이트 (버프/디버프, 상태 변화)
        - SimulationActionMessageGroup: 액션 메세지 (네트워크 패킷) 처리
        - SimulationUnitControlGroup: 센티널 (유닛 조종 커서)의 제어 상태 변화 및 유닛 명령
        - SimulationPreAnimationGroup: 결정론 애니메이션 재생 이전 AI 상태 제어
        - SimulationAnimationGroup: 결정론 애니메이션 재생, 이벤트 출력
        - SimulationPostAnimationGroup: 결정론 애니메이션 이벤트 소비자
        - SimulationProjectileGroup: 투사체 제어
        - SimulationDamageGroup: 전투 시뮬레이션 상태 업데이트

##### 1.1. 구조에 대한 고민

- ECS 시스템은 UpdateBefore/UpdateAfter 과 같은 Attribute를 통해 각 시스템간의 업데이트 순서를 고정시켜야 함
- 여러 작업자가 동시에 협업하는 상황, 시스템의 개수가 늘어나면서 시스템 간의 업데이트 순서를 트래킹하기 힘들어지고 문제가 발생하기 쉬움
- 따라서, 큰 서브그룹을 만들어 업데이트 순서를 고정해두고, 추가되는 시스템은 해당 그룹 안에서의 순서만 조정하면 깔끔함

##### 2. ECS 시스템 세부 동작 구현

- ECS 레이어에서 동작 중인 시스템의 약 87%를 구현
- 유닛을 구성하는 각종 컴포넌트, 유닛 팩토리 구조, 유닛 소환 및 AI, 로컬 회피, 이동, 공격 등 RTS 유닛의 필수 요소 구현
- 컴포넌트는 UnitAIStateData, UnitAIMoveData, UnitAITargetData 등으로 분리하여 데이터의 의도를 한 눈에 보여줄 수 있게 함
- 기존 스태틱데이터를 이용하여 초기화하던 데이터는 UnitBlobStoreSystem에서 BlobBuilder를 통해 BlobAssetReference로 저장
    - 동일한 데이터의 메모리 중복 사용을 낮출 수는 있으나, 랜덤 메모리 액세스여서 최대한 사용을 지양
- 로컬 회피는 기존 프로토타입에선 RVO를 사용하였으나, Boids/Steering/ORCA 등을 테스트해본 결과 가장 회피 효과가 좋았던 리스크 판별 후 경사하강을 이용한 탈출 벡터 계산방식을 이용함.
    - 이웃 유닛 탐색에는 Z->X->유니크 ID순으로 정렬된 공간 해시와 K-nearest-neighbor를 활용하여 유닛에서 가까운 이웃들을 탐색
    - 다만 교전 상황이나 다수 유닛 컨트롤 중 유닛들이 크게 회피하는 경향성이 생겨 해결법을 R&D 중 이었음 (리스크가 임계값을 넘으면 강한 회피 모드 돌입, 임계값보다 낮은 어떤 값보다 낮아지면 강한 회피 모드 해제 같은 방식)
- 게임 특성 상 유닛 단일 컨트롤이 아니라 동일 타입 혹은 전체 유닛 동시 컨트롤이기 때문에 스타크래프트 2 스타일 단체 이동 로직을 개발함 (현재 포메이션 유지한 채로 이동, 막히면 해당 위치에서 홀드)
- 애니메이션의 경우 스태틱 데이터에 틱-이벤트ID 구조를 만들고, ECS에서 Idle<->Move, Attack, Skill, Fusion과 같은 애니메이션을 재생할 수 있는 간단한 상태 머신을 제작
- EcsPresenter 라는 순수 C# 구조체를 정의하고 ECS<->GO 레이어 간의 브릿지로 활용하여 유닛 비주얼/FX 프리팹 생성, Animator 제어 등에 활용함
- 기타 특별 게임모드들을 위해 스탯 버프/디버프, 상태이상 시스템 구현
    - 스탯과 상태이상은 유닛 엔티티의 버퍼에 Request 구조체를 추가하면 해당 시스템들이 검색하여 스탯과 상태이상 구조체를 버퍼에 추가하는 방식으로 구현
    - 해당 방식을 통해 버프와 상태이상의 적용 시점을 통일시키고, Job으로 묶어서 병렬처리할 수 있었으나 적용이 1틱 늦어진다는 단점이 있음
<br>
<details>
<summary>상세 구현 목록</summary>

<ul>
  <li>SimulationBootstrapGroup
    <ul>
      <li>TeamAssignmentSystem</li>
      <li>UnitSpawnSystem</li>
      <li>BuildingSpawnSystem</li>
      <li>TransformPrevSnapshotSystem</li>
    </ul>
  </li>
  <li>SimulationStatsGroup
    <ul>
      <li>BuffRequestSystem</li>
      <li>BuffExpireSystem</li>
      <li>StatRecalcSystem</li>
      <li>StatusRequestSystem</li>
      <li>StatusExpireSystem</li>
      <li>StatusFlagsRecalcSystem</li>
      <li>SentinelFreezeStateSystem</li>
    </ul>
  </li>
  <li>SimulationActionMessageGroup
    <ul>
      <li>AutoAttackNeutralActionMessageSystem</li>
      <li>SentinelActionMessageySystem</li>
    </ul>
  </li>
  <li>SimulationUnitControlGroup
    <ul>
      <li>SentinelMoveTypeUpdateOnFollowSystem</li>
      <li>SentinelGroupControlSystem</li>
      <li>SentinelMoveTypeUpdateSystem</li>
      <li>SentinelPathPreferenceSystem</li>
      <li>SentinelAttackMoveSystem</li>
      <li>SentinelMoveInputSystem</li>
      <li>GroupMoveOrderEnqueueSystem</li>
      <li>UnitGroupMoveOrderSystem</li>
    </ul>
  </li>
  <li>SimulationPreAnimationGroup
    <ul>
      <li>UnitFusionSystem</li>
      <li>RallyPointApplySystem</li>
      <li>UnitAISystems</li>
      <li>UnitAIAttackSystem</li>
      <li>AttackStartFromRequestSystem</li>
      <li>UnitAirCollisionSyncSystem</li>
      <li>UnitDynamicCostBuildSystem</li>
      <li>UnitAIRepathDecisionSystem</li>
      <li>UnitAvoidanceSpatialHashBuildSystem</li>
      <li>UnitAILocalAvoidanceSystem</li>
      <li>UnitAIMoveSystem</li>
      <li>UnitMoveSettleSystem</li>
    </ul>
  </li>
  <li>SimulationAnimationGroup
    <ul>
      <li>UnitAnimationPlaySystem</li>
      <li>UnitAnimationTickSystem</li>
      <li>UnitAnimationLoopSystem</li>
      <li>UnitAnimationFxEventPresentationSystem</li>
      <li>UnitAnimationPresentationRequestSystem</li>
    </ul>
  </li>
  <li>SimulationPostAnimationGroup
    <ul>
      <li>AttackResolveFromAnimationEventSystem</li>
      <li>UnitHeightInterpolationEventSystem</li>
    </ul>
  </li>
  <li>SimulationProjectileGroup
    <ul>
      <li>ProjectileSpawnSystem</li>
      <li>ProjectileHomingSystem</li>
    </ul>
  </li>
  <li>SimulationDamageGroup
    <ul>
      <li>UnitSpatialHashBuildSystem</li>
      <li>AttackApplyHitSystem</li>
      <li>UnitDeathCleanupSystem</li>
      <li>SatelliteAutoDeactivateSystem</li>
      <li>SentinelFreezeApplySystem</li>
    </ul>
  </li>
</ul>

</details>
<br>

##### 3. ECS 전환 성과

- 기존 배틀로얄 모드에서 발생하던 원인 미상의 디싱크 해결
- 리팩토링 이후 디싱크 이슈 0건
- 동일 스펙 기기에서 타겟 프레임 내에 존재할 수 있는 유닛 수 대폭 상승 (40기→100기)
- PC 기준 전체 시스템 연산에 프레임 당 약 4ms 소요, 30fps 타겟 기기인 A05S에서 9ms 소요

##### 4. 아쉬운 점

- 결정론 안정성을 우선해 병렬화 범위를 핵심 경로로 제한함. 그 결과, 대규모 교전 구간에서 CPU 여유를 추가 확보할 여지가 남았음
- 리팩토링 기간이 2026.01.13~2026.02.14(약 33일)로 짧아 구조 안정화 중심으로 우선 적용했고, 일부 일반화 작업은 후속 과제로 남겼음
- 후속 개선으로는 시스템별 처리 데이터 명세를 먼저 정리하고, Job 전환 가능 구간을 식별해 read/write phase 기반 병렬화 확대와 길찾기 병렬화를 진행할 수 있음