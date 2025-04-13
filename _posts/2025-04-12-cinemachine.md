---
title: >
    Unity Cinemachine 3 탐구
tags: [Unity]
style: border
color: primary
description: >
    Unity 6에 맞춰 업그레이드된 Cinemeachine 3를 샘플 씬 위주로 탐구
---

![클리어 샷](assets/cinemachine.png)

항상 상용 유니티 게임 에셋들을 살펴보다 보면 튀어나오는 CM vcam 오브젝트들, 어떤건지 최근에야 알았습니다. 바로 유니티에서 제공해주는 강력한 시네마틱 도구 Cinemachine의 Virtual Camera 오브젝트들이었는데요, 이 시네머신이 유니티 6를 맞아 Cinemachine 3으로 업데이트 되었다하여 한 번 살펴봤습니다.<br>

![카메라 명칭](assets/cinemachine2.png)

Cinemachine 3의 가장 큰 특징은 비직관적이었던 카메라 명칭들의 간소화와, 카메라 - 컴포넌트 - 익스텐션의 확실한 모듈화라고 하는데요. 확실히 버츄얼 카메라라는 이름은 실제 렌더링에 사용되는 카메라가 아니라는 의미가 어느정도 있긴 했지만 그렇다고 다른 카메라가 버츄얼이 아닌건 또 아니고... 조금 애매하긴 했죠. 단순하게 시네마틱 카메라로 변경되었습니다.<br>
또한, 패키지를 설치하면 샘플 탭에서 따로 샘플 씬들도 임포트할 수 있는데 저는 이 씬들을 위주로 Cinemachine 3를 한 번 살펴보고자 합니다.

### ClearShot

![클리어 샷](assets/cinemachine3.png)

클리어 샷 카메라는 차일드로 지정된 카메라 중 현재 샷 퀄리티가 가장 좋은 카메라를 자동적으로 선택하는 카메라입니다. 만약 샷 퀄리티가 준수한 카메라가 여러 대 발견된다면, 그 때는 우선 순위가 가장 높은 카메라가 선택됩니다.<br>
샘플 씬은 정면 카메라를 가장 높은 우선순위로 하여, 좌우로 움직이며 장애물에 가려지는 캐릭터가 항상 화면에 잡히도록하는 클리어 샷 카메라 구성을 보여줍니다.

![정면 카메라](assets/cinemachine4.png)

장애물에 정면샷이 가려지지 않는 경우 이렇게 정면 카메라가 작동되게 되고, 위의 사진처럼 장애물 뒤에 숨어있는 경우 오버헤드 카메라 혹은 비하인드 카메라 중 오버헤드 카메라를 위주로 "클리어 샷"을 포착하는 카메라가 작동되게 됩니다.<br>

### Cutscene

![클리어 샷](assets/cinemachine5.png)

Cutscene 씬은 Orbital Follow로 설정되어 유저의 인풋을 받아 캐릭터 주위를 회전하게 되는 Freelook 카메라와 타임라인을 통해 제어되는 3개의 일반 카메라로 구성되어 있습니다.<br>
보물 상자에는 Cinemachine Trigger Action 스크립트가 달려있어 캐릭터가 상자 주변에 다가가면 Open Chest Timeline 타임라인이 실행되고, 3단계에 걸쳐 상자에 줌인-진동-줌아웃의 컷씬이 재생되게 됩니다.<br>
타임라인에 시네머신 샷 트랙이 직접 삽입되어 있어서 카메라간의 블렌딩이 자연스럽게 이루어지는 것이 인상적입니다.

### FlyAround

![날으는 오브젝트](assets/cinemachine6.png)

FlyAround 씬에는 Cinemachine Input Axis Controller 컴포넌트에 New Input System의 액션을 연결하고 Unity.Cinemachine.IInputAxisOwner 인터페이스를 상속하여 인풋을 읽어서 날아다니는 오브젝트와 해당 오브젝트를 따라다니는 카메라로 구성되어 있습니다.<br>
카메라 연출 뿐만 아니라 기본적인 3D 공간에서의 조작까지 시네머신이 알아서 처리해준다는게 마음에 드네요.

### FreeLook Deoccluder

![충돌 감지](assets/cinemachine7.png)

이 씬에서는 Cinemachine Deoccluder 를 활용하여 충돌 대상 레이어를 설정하고, 충돌이 감지되면 카메라를 플레이어쪽으로 가깝게 당겨주는 기능을 시연하고 있습니다.<br>
이전 버전에서 Cinemachine Collider를 이용하면 비슷한 기능을 구현할 수는 있었지만, 피사체가 카메라에 담기는 것을 보장하지는 않았던 것 같은데 Deoccluder는 샷 퀄리티 판별을 통해 LoS를 보장해준다고 합니다. 이런 기능이 필요없다면 Cinemachine Decollider를 사용하면 될 것 같네요.

### FreeLook on Spherical Surface

![지구를 돌다](assets/cinemachine8.png)

비슷한 FreeLook 카메라지만 플레이어가 구체의 표면을 따라서 걷도록 만들고, Cinemachine Brain의 World Up Override를 이용하여 월드 업을 플레이어 업으로 덮어씌우는 방식으로 플레이어의 각도에 대응하는 카메라를 구현한 모습입니다.

### Impulse Wave

![지진 발생](assets/cinemachine9.png)

Cinemachine Impulse Source를 통해 진동을 발생시키고, 카메라나 오브젝트에 (External) Impulse Listener 스크립트를 장착하는 방식으로 진동이 퍼져나가면서 오브젝트와 카메라를 떨리게하는 효과를 구현한 씬입니다. 이런 기능을 활용한다면 폭발이 일어난다던지, 거대 생명체의 발울림 같은 것을 표현하는데에 정말 편리할 것 같습니다.

### Lock-on Target

![락온](assets/cinemachine10.png)

Cinemachine Trigger Action이 발동되면 캐릭터를 단순 Follow하고 보스를 Look At 하는 카메라로 전환되도록 구성된 씬입니다.

### MixingCamera

![믹싱 카메라](assets/cinemachine11.png)

자동차의 현재 속도에 비례하여 FOV와 위치값이 다른 두 카메라가 믹싱되도록 구성된 씬입니다. 확실히 빨라질 때 FOV 값이 늘어나면서 카메라가 뒤로 이동하니 더 속도감있고 재밌는 연출이 되어 좋은 것 같습니다.

### RunningRace

![리더 카메라](assets/cinemachine12.png)

시네머신의 확장성과 응용 가능성을 확실히 보여주는 씬인 것 같습니다. 정말 특이한게 일단 달리기 주자들부터 실제로 달리는게 아니라 Cinemachine Spline Cart 컴포넌트를 이용하여 Spline 위를 돌리 카메라처럼 달리도록 구현하였습니다.<br>
리더 카메라도 분명 클리어 샷 카메라는 샷이 잘 나오는 카메라가 사용되는 구조라고 했는데, 알고보니 IShotQualityEvaluator 인터페이스를 상속하는 별개의 스크립트를 이용하여 1등 주자가 베스트 샷이라는 결과값만 전달하는 방식으로 구현되어 있습니다.<br>
![스플릿 스크린](assets/cinemachine13.png)
그룹 프레이밍 카메라는 Cinemachine Group Framing 익스텐션을 이용하여 타겟 그룹 (주자들과 태양)이 모두 한 화면 안에 담기도록 자동 판별하여 샷을 담도록 구성되어 있습니다.

### SplitScreenCar

![스플릿 스크린](assets/cinemachine14.png)

채널 마스크를 이용하여 카메라 두 대가 각각 다른 시네머신 카메라의 정보를 받아오게 만들고, 뷰포트 아웃풋을 반으로 잘라 옛날 2인용 콘솔게임 느낌을 낸 씬입니다.

### ThirdPersonWithAimMode

![TPS](assets/cinemachine15.png)

Third Person Follow 타입의 카메라 두 대와 CinemachineCameraManagerBase 를 상속받는 Aim Camera Rig 라는 커스텀 카메라 매니저를 이용하여 3인칭 슈팅과 정조준을 구현한 씬입니다. Aim Camera Rig는 ChooseCurrentCamera 함수를 오버라이드하여 정조준 입력이 들어오는 경우 에임 카메라를, 정조준이 아닌 경우 프리룩 카메라를 사용하도록 하고 있습니다. 총알에 오브젝트 풀링도 구현되어 있고 굉장히 건실한 씬이네요...

### ThirdPersonWithRoadieRun

![TPS Run](assets/cinemachine16.png)

이전 TPS 씬에 State-Driven 카메라를 더해서 애니메이터가 달리기 상태에 들어가면 달리기 카메라가 발동되도록 구성된 씬입니다. 이걸 활용하면 캐릭터가 달릴 때 FOV 효과를 구현하는데에 정말 편리할 것 같습니다.<br>
<br>
이것으로 Cinemachine 3에 포함된 모든 샘플씬과 그 구현을 다 살펴보았습니다.<br>
코드 한 줄 쓰지 않아도 이렇게 상황에 맞는 카메라를 재생시킬 수도 있고 카메라 간 블렌딩도 자연스럽게 조절할 수 있다는건 정말 큰 메리트인 것 같습니다.<br>