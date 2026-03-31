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
- 1vs1만 지원하던 구조에서 다인 멀티플레이, 팀전으로 스펙 업그레이드
- 신규 모드 (배틀로얄, 넥뿌) 프로토타이핑
- FX 툴 및 FX 바이너리 데이터 구조 제작
- 저사양 최적화 (ARM Profiler, Unity Profiler, RenderDoc 활용)

# 3. 업무 상세

## DOTS + ECS 리팩토링

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

- ECS 에서는 각 시스템을 업데이트 그룹으로 묶어서 관리 가능, 그룹의 서브그룹도 정의 가능
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

##### 1.1 구조에 대한 고민

- ECS 시스템은 UpdateBefore/UpdateAfter 과 같은 Attribute를 통해 각 시스템간의 업데이트 순서를 고정시켜야 함
- 여러 작업자가 동시에 협업하는 상황, 시스템의 개수가 늘어나면서 시스템 간의 업데이트 순서를 트래킹하기 힘들어지고 문제가 발생하기 쉬움
- 따라서, 큰 서브그룹을 만들어 업데이트 순서를 고정해두고, 추가되는 시스템은 해당 그룹 안에서의 순서만 조정하면 깔끔함

##### 2. ECS 시스템 세부 동작 구현

- ECS 레이어에서 동작 중인 시스템의 약 87%를 구현

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
      <li>SeninelActionMessageySystem</li>
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
