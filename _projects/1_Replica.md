---
name: 2D 플랫포머 포트폴리오
tools: [C#]
image: assets/replica.png
description: 유니티 C#을 이용한 2D 플랫포머 액션 게임. Art by Aduare
---

# Replica
{% include elements/video.html id="kP5UOgp4Eb0" %}
팀원들의 일정 문제로 개발 중단된 프로젝트.<br>
아트 담당 1명 ([Aduare](https://x.com/aduare_rp))과 제가 메인 프로그래머로 작업하여 큰 틀을 구성한 후에 버그 픽스 및 추가 작업을 위한 프로그래머를 10월경 구했지만 결국 완성까지는 이어지지 못했습니다.

<p class="text-center">
{% include elements/button.html link="https://github.com/jarari/Replica" text="프로젝트 GitHub" %}
</p>

## 핵심 구현 요소
<br>
### 1. 빠르고 경쾌한 액션

![전투 프로토타입](assets/replica_action_demo.gif)

검, 총, 주먹 3가지 무기의 콤보시스템과 피격 효과, 카메라 쉐이크를 통해 가볍고 빠르면서도 묵직한 액션을 추구하였습니다.<br>
<br>
### 2. 배경 시뮬레이션

![배경 시뮬레이션](assets/replica_scenarysim_demo.gif)
![배경 시뮬레이션2](assets/replica_scenarysim.png)

배경의 길이, Z좌표 등을 이용하여 카메라의 위치에 따라 3D 월드처럼 느껴지도록 배경을 이동하는 시스템을 구현하였습니다.<br>
<br>
### 3. 레벨 동적 생성

![레벨 블록과 패턴](assets/replica_mapdata.png)
Scene에 배치된 레벨 오브젝트들을 json 데이터로 내보내고, 실제 게임 플레이 시에는 해당 Scene이 아닌 맵 데이터를 이용해서 지형을 재구성하는 시스템을 구현하였습니다.<br>
기획된 내용상으로는 마비노기 던전처럼 던전 구획을 블럭화하여 랜덤 생성하는 방식을 이용하여 플레이어에게 좀 더 독특한 경험을 제공하기 위함입니다.<br>
<br>
### 4. 최적화 및 확장성 보장

![데이터 예시](assets/replica_dataexample.png)
최적화를 위해서 Instantiate 기능 대신 오브젝트 풀링을 통한 동적 할당/해제 시스템을 구현하였고, 게임을 구성하는 모든 데이터는 확장성을 위해 상속 관계와 외부 json 데이터를 통해 해당 오브젝트에 필요한 부분만 프로그래밍하여 사용할 수 있도록 구현하였습니다.<br>