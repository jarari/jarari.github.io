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
- 프로젝트 보안 정책상 일부 구현 세부는 비식별화해 기술했습니다.

# 2. 담당 업무 개요

- DOTS + ECS 구조로 리팩토링
- 저사양 최적화 (ARM Profiler, Unity Profiler, RenderDoc 활용)
- FX 툴 및 FX 바이너리 데이터 구조 제작
- 신규 모드 (배틀로얄, 넥뿌) 프로토타이핑
- 1vs1만 지원하던 구조에서 다인 멀티플레이, 팀전으로 스펙 업그레이드

# 3. 업무 상세

## 1. DOTS + ECS 리팩토링

### 작업 배경

- 10인 배틀로얄 모드가 추가되면서 한 클라이언트 내 처리해야 하는 유닛의 규모가 대폭 상승함 (기존 80 → 변경 후 200)
- 아트스타일 리워크로 인해 타겟 모바일 기기에 대한 부하가 심해짐
- 프로젝트 프로토타입이 MonoBehaviour를 과도하게 많이 사용하고 있어 코드 흐름을 파악하기 어렵고, 이로 인해 디싱크가 발생하였음
- 기획팀에서 프로토타입에 구현된 유닛들의 움직임 (특히 길찾기, 로컬 회피)에 대해 플레이 경험이 좋지 않다고 피드백

### ECS 도입 사유

- 메모리에 엔티티가 archetype/chunk 단위로 순차적 저장되기 때문에 캐시 친화적, 대량 시뮬레이션에 유리
- Job System을 활용하여 데이터 참조→상태 변경의 흐름을 자연스럽게, 그리고 병렬적으로 처리될 수 있도록 구현하면서 Burst를 통한 추가 최적화로 성능 향상
    - 모바일 기기에서 FixedPoint 연산의 비용, 부가적으로 발생하는 CPU 코스트를 멀티쓰레딩으로 우회하려는 목적 포함
- 코드의 흐름을 일직선 시스템으로 단순화하여 빠른 디싱크 원인 파악 및 해결
- 컴포넌트/시스템으로 분리된 구조를 통해 명확한 목적 표시, 코드 가독성 향상
- 게임 로직과 비주얼 오브젝트를 완전 분리하여 신규 유닛 구현, 유닛 스킨 적용 플로우 개선

### 작업 내역

#### 1. ECS 시스템 구조 정의

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

#### 2. ECS 시스템 세부 동작 구현

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
      <li>SentinelActionMessageSystem</li>
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

#### 3. ECS 전환 성과

- 기존 배틀로얄 모드에서 발생하던 원인 미상의 디싱크 해결
- 리팩토링 이후 디싱크 이슈 0건
- 동일 스펙 기기에서 타겟 프레임 내에 존재할 수 있는 유닛 수 대폭 상승 (40기→100기)
- PC 기준 전체 시스템 연산에 프레임 당 약 4ms 소요, 30fps 타겟 기기인 A05s에서 9ms 소요

#### 4. 아쉬운 점

- 결정론 안정성을 우선해 병렬화 범위를 핵심 경로로 제한함. 그 결과, 대규모 교전 구간에서 CPU 여유를 추가 확보할 여지가 남았음
- 리팩토링 기간이 2026.01.13~2026.02.14(약 33일)로 짧아 구조 안정화 중심으로 우선 적용했고, 일부 일반화 작업은 후속 과제로 남겼음
- 후속 개선으로는 시스템별 처리 데이터 명세를 먼저 정리하고, Job 전환 가능 구간을 식별해 read/write phase 기반 병렬화 확대와 길찾기 병렬화를 진행할 수 있음

## 2. 저사양 최적화

### 작업 배경

- 30fps 타겟 기기인 A05s에서 유닛 한 기도 뽑지 않은 상태인데도 40fps, 1vs1 최대 유닛수인 80기 도달 시 15fps, 교전 시작 시 렉으로 인해 다운되는 현상 발생
- 기존 프로토타입 코드에서 잘못 짜여진 로직, 전반적인 드로우콜, 렌더 서브미션 순서, Early ZS, 셰이더 연산량, UGUI 캔버스 리빌드에 대한 전체 재점검 및 개선 필요

### 작업 내역

#### 1. 과도한 메모리 대역폭 사용 개선

- Unity Profiler의 Rendering 섹션과 Frame Debugger로 분석한 결과 전장의 안개, 미니맵에 1024x1024 사이즈 RT가 사용되고 있었으며, 풀스크린 패스는 잘못된 설정으로 Color 텍스처가 풀 해상도로 1회 복제되고 있었음
- [Using DOTS to optimize GameObject gameplay](https://www.youtube.com/watch?v=ZkvK0mX-id4) 에서 소개된 내용을 기반으로 Job 시스템/Burst를 활용하여 병렬적으로 유닛들의 위치를 수집하고 미니맵 위의 좌표로 변환하여 업데이트하는 구조로 변경을 제안
- [A Story of Fog and War](https://www.riotgames.com/en/news/story-fog-and-war) 의 내용을 기반으로 전장의 안개 텍스처 크기 축소와 픽셀 데이터 쓰기 방식 개선 제안
- 해당 변경으로 매 프레임 발생하던 약 10MB의 텍스처 read/write를 제거

#### 2. CPU 병목 원인 파악 및 개선

![ARM Profiler](assets/kong82_1.png "ARM Profiler") | ![Android Studio](assets/kong82_2.png "Android Studio")

- ARM Profiler와 Android Studio에서 분석한 내용을 바탕으로 현재 게임이 CPU 병목 상태에 처해 있다고 판단
- CPU 병목의 주요 원인은 Unity Profiler을 통해 다음과 같다고 판단함
    - 드로우콜, 렌더 배칭 실패, URP Render Objects 기능의 레이어 분리로 인한 CPU 측의 렌더 서브미션 비용
    - SRP Batching이 불가능한 SkinnedMeshRenderer 들에 Animator 로 애니메이션을 적용하는 과정에서 생기는 부하
    - UGUI 최적화 실패로 인한 드로우콜 증가, 캔버스 레이아웃/그래픽 리빌드 비용
    - FixedPoint를 사용하는 시뮬레이션 로직과 길찾기 부하
    - 파티클과 사운드 풀링 로직 오류

##### 2.1. 렌더 서브미션 비용 개선

- CPU 측의 렌더 서브미션 비용을 낮추기 위해 Render Objects의 사용을 최소화하는 방향을 실험함
- 또한, Stacking을 활용하는 카메라를 전면 제거하고 UI 또한 Screen Space - Overlay로 변경하여 CPU 코스트 감소, 프레임 증가를 관측함
    - 이 과정에서 UI 디자이너분이 만드신 일부 UI가 빨갛게 변하는 현상이 발견되었는데, 이는 Shader Graph의 Material이 Sprite Unlit으로 설정되어 UV의 해석에 불일치가 발생하였기 때문임. 이는 URP 17부터 추가된 Canvas Material을 사용하는 것으로 해결
    - 다만, Canvas Material의 경우 Vertex Context의 Position을 사용할 수 없기 때문에 정점 변환이 불가능함. 이 경우 HLSL로 직접 셰이더 작성이 필요하였음

##### 2.2. Animator 비용 개선

- SkinnedMeshRenderer + Mecanim 을 대량 생성한 경우에 생기는 프레임 드랍의 경우 널리 알려져 있으나, 성능이 PC에 비해 월등히 빈약한 모바일, 특히 타겟 저사양 기기에서는 크게 도드라졌음
- 최신 중국 게임들이 애니메이션의 업데이트 주기를 제한하던 것에서 착안하여 PlayableAPI를 활용하여 유닛들의 Animator들을 하나의 PlayableGraph에 묶고 15hz/30hz/60hz로 업데이트 주기를 제한해 봄
    - 단순 클립 하나를 재생하는 경우에는 업데이트 주기 제한이 꽤 효과적이었으나, 클립 간 보간을 위해 AnimationMixerPlayable을 활용하는 순간부터 15hz로 설정하였을 때 잃게 되는 비주얼적 손해에 비해 성능 이득이 크지 않았음
    - 다만, 기존 프로토타입 구조에서 StateMachineBehaviour 스크립트를 통해 구현되어 종종 애니메이션 레이어 설정 오류를 일으켰던 모드 변경식 유닛들이 결정론적 애니메이션 재생 -> 대응되는 클립을 진행 상태에 1:1 싱크하는 방식으로 구현했을 때 완벽하게 동작하는 것을 발견하여 해당 유닛들은 이 방식을 사용하도록 유지
- [Unite Austin 2017 - Massive Battle in the Spellsouls Universe](https://www.youtube.com/watch?v=GEuT5-oCu_I) 에서 대량 시뮬레이션을 위해 제안된 Indirect Instanced Rendering + Vertex 셰이더를 통한 GPU Skinning 구현
    - SMR을 사용하는 것이 아니라, Graphics API의 RenderMeshIndirect 기반 인스턴싱 렌더링으로 유닛을 직접 그림. 이 과정에서 같은 Mesh/Material/Animation 등 성격별로 Batch를 묶고, 인스턴스별 위치/회전/색상/클립 인덱스/클립 시간/블렌드 가중치는 MaterialPropertyBlock에 바인딩된 StructuredBuffer 로 전달
    - 처음에는 애니메이션 클립을 매 프레임 각 bone의 local transform matrix 를 저장하였으나, SMR가 여러 개로 쪼개져 있는 유닛의 경우 메인으로 설정한 SMR에서 특정 본에 대해 weight가 0인 경우 해당 본이 누락되는 문제가 있었음. 해당 문제를 해결하기 위해 사용자가 직접 기준점으로 사용될 bone을 선택하고 SMR을 모두 지정하면, 지정된 모든 SMR에서 bone array를 중복 없이 추출/샘플링 후 실제 skinning 단계에서는 bone index를 통한 remapping 과정을 거친 후 RendererLocalFromCanonicalRoot * CanonicalMatrix * BindPose 로 CPU에서 본의 위치를 계산한 뒤 GPU에서 본 웨이트에 의한 정점의 변환을 계산함

##### 2.3. UGUI 개선

- 캔버스 UI에서는 draw order 사이에 material, texture, clipping 상태가 다른 요소가 끼어들거나, 서로 다른 shader/material 또는 다른 material instance가 사용될 경우 batching이 분할되어 draw call이 증가할 수 있음
    - UI 디자이너와 협업하여 복잡한 계층 구조를 단순화하고, UI 간 겹침을 최소화했으며, shader를 가능한 범위에서 통일함
    - BaseMeshEffect를 활용해 정점의 uv1/uv2로 설정값을 전달하도록 수정하여, 동일 shader/material을 유지하면서 material instance 생성을 줄였음
- 실제 값 변경 없이 UI material/vertex 상태를 매 프레임 dirty 상태로 표시하는 코드로 인해 불필요한 Graphic Rebuild가 발생할 수 있음
    - 값을 캐싱한 뒤, 변경이 발생한 경우에만 대입하도록 수정하여 불필요한 dirty 마킹을 줄였음
- 캔버스 내 UI의 모양, 재질, 텍스트, 이미지, 레이아웃 변경은 Canvas Rebuild의 원인이 되며, 변경 빈도가 높은 요소가 큰 Canvas에 포함되어 있을 경우 리빌드 비용이 커질 수 있음
    - 업데이트 주기가 다른 UI 요소들을 기준으로 Canvas를 분리해 Rebuild 범위를 축소함
    - 단, Canvas 분리는 batching 감소로 이어질 수 있어 실제 효과를 프로파일링하며 적용함
- Animator로 제어되는 UI 요소는 애니메이션 상태 유지 과정에서 지속적으로 프로퍼티 변경이 발생해 Canvas Rebuild 비용의 원인이 될 수 있음
    - 신규 UI 연출은 DOTween 중심으로 가이드를 정리했고, 기존 작업물은 수정 비용을 고려해 일부 구간에 Legacy Animation을 활용하여 임시 대응함

#### 3. 낮은 Early ZS Test Rate

![Low Early ZS Test Rate](assets/kong82_3.png "Low Early ZS Test Rate")

- Android 기기에서 프레임 드랍이 특히 심하였는데, 처음에는 메모리 대역폭 차이를 의심하였으나 ARM Profiler로 Mali 기기를 프로파일링 하는 과정에서 Early ZS Test Rate가 낮게 잡히는 현상을 관측
    - RendorDoc을 통해 렌더 순서를 분석하여 보았으나, Opaque 내에서는 front-to-back 드로우 순서가 잘 지켜지는 것을 확인함
    - 이후 인게임 상에서 Hierarchy를 제어할 수 있는 플러그인을 설치하고 의심가는 항목들을 하나씩 비활성화 하는 방식으로 테스트

![Fog off](assets/kong82_4.png "Fog off")

- 전장의 안개를 제거하자 Early ZS Test Rate가 거의 100%에 도달하는 현상을 관측함. 구현을 확인해본 결과, 맵 전체를 덮는 SpriteRenderer 오브젝트를 두고 해당 스프라이트에 블랙&화이트 마스킹 픽셀 데이터를 세팅하는 구조로 되어 있었음
    - Transparent 오브젝트가 화면 전체를 덮고 있기 때문에 Early ZS가 depth를 판정할 수 없는 상태
- 해당 오브젝트를 완전히 제거하고 배경과 유닛 셰이더에 안개 색상을 블렌딩하는 방식을 구상
    - Include 용으로 공통 셰이더를 제작하고, 월드 포지션을 맵 크기로 나누어 FogMap의 UV상에 매핑. 이렇게 하면 월드 좌표에 따른 [0-1] RGB값을 받을 수 있음
    - 배경, 유닛 셰이더의 최종 컬러값을 안개 색상과 FogMap 샘플링 값으로 블렌딩하여 최종 색상 결정
    - 변경 이후 Early ZS Test Rate 100% 달성, A05s 기준 프레임 4 증가

#### 4. 전장의 안개 개선

![Fog Improvement](assets/kong82_5.png "Fog Improvement")

- 전장의 안개가 저해상도 마스크맵 기반으로 구현되어 있어 외곽선이 픽셀화되어 보였고, 이를 완화하기 위해 기존에는 스크립트에서 feathering 로직을 수행하고 있었음
    - 샘플링용 FogMap을 128x128 → 256x256으로 Graphics.Blit 기반 업스케일 후, 셰이더에서 가우시안 블러를 적용하는 방식으로 변경하여 외곽선 품질을 개선함
    - CPU에서 처리하던 feathering 연산을 GPU 패스로 이전해 CPU 부하를 줄이고, 품질과 성능을 함께 개선함
    - 최대 전장 크기가 200x200 수준이었기 때문에, 품질 개선 효과와 렌더링 비용을 함께 고려해 FogMap 해상도는 256x256으로 제한함

#### 5. 기타 로직 오류 및 CPU 오버헤드 개선

- 월드에 수십~수백개 존재할 수 있는 체력바, 빌보드 스크립트 등 MonoBehaviour.Update 혹은 LateUpdate를 이용하던 스크립트들을 중앙화하여 호출 분산 비용을 줄이고, 조건별 업데이트 제어가 가능한 구조로 개선함
    - 단일 항목의 비용은 크지 않았지만(약 0.5ms), 저사양 환경에서는 누적 오버헤드와 관리 복잡도를 줄일 필요가 있다고 판단해 구조를 개편함
    - 개선 후 비용 0.2ms로 감소
- 프로토타입 단계에서 render scale을 1920 기준 해상도로 계산하도록 구현되어 있었고, 이로 인해 저해상도 기기에서는 오히려 업스케일링이 적용되는 문제가 있었음
    - 임시 대응으로 저사양 기기에는 0.75, 그 외 기기에는 1.0을 적용하도록 수정했으며, 이후에는 그래픽 옵션으로 사용자 선택이 가능하도록 확장할 계획이었음
- 전투 시 발생하는 프레임 드랍 개선
    - 프로토타입 코드에서 Enum.ToString, LINQ의 사용, 잘못된 FX 라이프 사이클 설정으로 인해 FX 재생 시/사운드 재생 시 GC Alloc과 CPU 스파이크가 발생함
        - 관련 로직을 전면 수정하고, FX 라이프사이클은 실제 재생 시간에 맞춰서만 유지되도록 기준을 정리해 작업자들에게 공지함
    - 공격, 사망 등 빈번하게 재생되는 효과음이 Streaming으로 설정되어 있어, 재생 시점마다 불필요한 로드 비용이 발생하고 있었음
        - 반복 사용되는 효과음은 Compressed In Memory로 Load Type을 변경하여 재생 시 로드 비용을 줄임
    - 사운드의 동시 재생 수 상한이 없어 대규모 전투 시 사운드 믹싱으로 인한 CPU 비용 발생
        - 우선 프로젝트 Audio 설정에서 Max Virtual Voices를 512→64, Max Real Voices를 32→16으로 조정해 비용을 완화했으며, 이후에는 풀링 개수와 동시 재생 정책 자체를 제한하는 방향으로 개선할 계획이었음
    - 해당 작업 후 GC 완전 제거, 고사양 기기에서는 100vs100 전투도 큰 프레임 드랍 없이 처리 가능해짐

### 성과

![Frame Debugger](assets/kong82_6.png "Frame Debugger")

- 전투 시 고사양 기기에서도 발생하던 프리징/렉 전면 해소
- UGUI 드로우콜 89→35로 감소
- 전체 테스트 기기에서 15~20 프레임 상승
- Y700 2세대
    - 상시 프레임: 110→120
    - 40vs40 생산 시 프레임: 90→115
    - 40vs40 전투 시 프레임: 30→90
- A05s
    - 상시 프레임: 35→55
    - 40vs40 생산 시 프레임: 15→30
    - 40vs40 전투 시 프레임: 1→15

## 3. FX 툴 및 FX 바이너리 데이터 구조 제작

## 4. 신규 모드 (배틀로얄, 넥뿌) 프로토타이핑

### 작업 배경

- 1vs1 모드만으로는 캐주얼 층을 잡기 힘드므로 타 게임, 스타크래프트 유즈맵에서 영감을 받은 다양한 모드를 제작

### 요구 스펙

- 빠른 프로토타이핑, 단체 테스트
- 완벽한 안정성, 데이터 구조보다는 플레이 흐름에 집중

### 구현 방식

- 코딩 AI 에이전트를 활용하여 기획팀에서 요구한 요소들을 모두 플레이 테스트 가능한 수준으로 빠르게 프로토타이핑
    - 2025.07.09 입사하여 2025.07.18 에 기획 회의, 2025.07.25 에 배틀로얄 게임모드 시스템 + 임시 맵 + 유닛 + 데이터를 포함한 프로토타입 빌드 전달
    - 2026.02.23 기획서 전달 받은 후 2026.02.25 저녁에 넥서스 부수기 게임모드 시스템 + 임시 맵 + 데이터 + 서버 매칭 API를 포함한 프로토타입 빌드 전달
- 유닛의 스펙에 자연스러운 조작 및 변형이 가능하도록 시스템 구조 개선
    - 기획팀에서 요구하는 "유닛 모드"가 일반적인 스펙 변경부터 공격 무기 변경, 일정 시간 불사, 스플래시 데미지 범위 증가와 같이 특수한 효과까지 광범위했기 때문에, 모드를 추가/제거 할 수 있는 API와 각 유닛이 들고 있는 순수 C# 객체를 정의하여 모드의 적용/필터링/지속 시간 경과를 트래킹 하였고, 모드 효과는 IModEffect 인터페이스를 상속받도록 하여 특수한 모드가 필요한 경우 해당 경로를 통해 적용되도록 함
    - 프로토타이핑 과정에서 기능이 대거 추가되거나 삭제되는 경우가 빈번하였지만, 해당 구조는 최대한 이후 신규 유닛 디자인이나 신규 모드에도 활용할 수 있도록 관리 주체와 외부 호출 경로를 명확히 하고, 효과를 쉽게 추가할 수 있게 하고 싶었음

### 한계

- ECS 리팩토링 스펙에 해당 기능이 포함되지 않아 ECS 유닛용은 구현되지 않았음

## 5. 다인 멀티플레이, 팀전으로 구조 스펙 업그레이드

### 작업 배경

- 프로토타입의 코드 구조는 1vs1 모드만을 상정하고 제작, 유닛 및 건물의 소유자 표기에 Faction 이라는 enum을 사용하여 확장성이 떨어짐
- 이후 배틀로얄과 넥서스 부수기 모드가 기획되면서 최대 10인 지원, 팀 개념의 필요성이 생김

### 요구 스펙

- 봇, 중립 유닛 구분 가능
- 최대 32인까지 상정
- 아군/적군 판별의 유연성
- 당장 테스트 가능해야 함

### 구현 방식

- 유닛에 OwnerId와 TeamId 항목을 int로 추가하고, OwnerId에는 로그인 한 유저의 id값을 사용하도록 세팅
    - 기존의 팩션 할당 플로우에 추가하여 흐름 유지
    - TeamId는 1vs1 모드, 리플레이에서도 동일하게 동작하여야 하기 때문에, 팀전이 필요한 모드에서만 세팅하는 방향으로 협의하고 팀이 세팅되지 않은 경우 헬퍼 함수가 유저 id를 그대로 반환하게 하여 OwnerId == TeamId가 되도록 함
    - 봇과 중립 유닛은 유저 id와 겹치지 않는 값을 미리 할당하여 필요한 경우 사용하도록 함
- 모든 구조를 한 번에 바꾸는 대신, 기존 팩션 클래스에 static 헬퍼를 추가하여 팩션<->유저 id를 변환해서 사용할 수 있도록 함
    - 자원, 건물 건설, 생산과 같이 1vs1에서만 사용되고 배틀로얄에선 필요하지 않은 기능은 후순위로 두어 안정성 보장

### 한계

- 동맹 설정/해제는 요구 스펙에 포함되지 않으므로 팀을 기준으로 적과 아군을 판별하도록 함. 추후 요구에 따라 팀 간 동맹 매트릭스를 만들어서 bit로 아군/적군 여부를 판별할 수 있음