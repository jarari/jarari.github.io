---
name: Uneducated Shooter
tools: [C++]
image: assets/uneducatedshooter.png
description: 폴아웃 4에 현대 택티컬 슈터의 요소를 추가하는 모드
---

# Uneducated Shooter

폴아웃 4는 2015년에 출시된 게임으로, 상당히 오래된 작품입니다. 또한, 슈터 요소보다는 롤플레잉 요소에 조금 더 치중되어 있는 게임이기도 합니다. 최근 플레이어들의 트렌드는 이와는 사뭇 다릅니다. 아에 Apex Legend, Valorant 같이 아케이드성을 강조한 게임이나 Rainbow Six: Siege나 Escape From Tarkov같이 극단적으로 현실적이고 전략적인 게임들이 유행하고 있죠. 이 트렌드에 맞춰 저는 폴아웃 4의 전략성을 강화시키는 모드를 제작하였습니다.<br>

<p class="text-center">
{% include elements/button.html link="https://github.com/jarari/UneducatedShooter" text="프로젝트 GitHub" %}
</p>
## 주요 기능
<br>
### 1. 카메라 및 총기 관성
{% include elements/video.html id="6k_rZS9hRXA" %}
[Bodycam](https://www.youtube.com/watch?v=OJtv52GuSWM) 이라는 게임에서 영감을 받은 저는 해당 게임과 비슷한 묵직한 느낌을 주면서도 게임 플레이에는 크게 방해되지 않을 정도의 카메라 무빙과 총기 모델 회전을 구현하는데에 집중하였습니다. 이미 폴아웃 4에는 additive animation의 블렌딩을 통해 카메라가 회전했을 때 1인칭 모델에 어느정도의 관성이 적용되고 있었습니다. 하지만, 굉장히 절제된 강도로 적용되어 있기도 하고 총기마다 애니메이션 스타일이 통일되어 있지 않아 그렇게 현실적으로 다가오지는 않았기 때문에 저만의 스타일로 이러한 시스템을 구현해보고자 했습니다.<br>
<br>
### 2. 좌우 기울이기

![타르코프 스타일](assets/uneducatedshooter_tarkov.webp "타르코프 스타일") | ![시즈 스타일](assets/uneducatedshooter_r6s.webp "레인보우식스: 시즈 스타일")

Rainbow Six: Siege, Escape From Tarkov 같은 택티컬 슈터 장르가 다른 슈터와 무엇이 가장 다른가 하면 아마 이 leaning and peeking 메카닉을 꼽을 수 있을 것 같습니다. 폴아웃 4에도 이러한 시스템이 1인칭에만 간단하게 구현이 되어있긴 했는데요, 정말 단순하게 1인칭 상태에서 정면에 엄폐물이 있을 때 정조준을 실행하면 플레이어의 이전 좌표를 기록해두고 엄폐물에서 가장 가까운 모서리 방향으로 플레이어를 강제 이동시키는 방식이었습니다. 저는 기존 기능을 완전히 비활성화 시키고, 1인칭과 3인칭 스켈레톤에 제가 컨트롤 할 수 있는 헬퍼 노드들을 런타임에 삽입하여 회전, 이동시키는 방식으로 좌우 기울이기를 구현하였습니다. 그리고 Escape From Tarkov같이 카메라와 총기 모델이 동시에 회전하는 방식의 경우 멀미를 느끼는 일부 유저들을 배려하여 Rainbow Six: Siege처럼 카메라의 각도는 그대로 유지되면서 총기 모델만 회전하는 기울이기 방식도 사용할 수 있도록 만들었습니다.<br>
<br>
### 3. 플레이어의 자세에 따른 콜리전 보정

베데스다 타이틀에서는 플레이어가 앉더라도 장애물 아래로 지나갈 수 없다는 문제가 있습니다. 게임 디자인적 선택인지는 알 수 없으나, 엔진 코드 상 이러한 부분에 대해 전혀 처리를 해주지 않고 있다는 것을 알 수 있었습니다. 저는 해당 문제를 해결하기 위해 현재 플레이어의 head 노드와 pelvis 노드, 그리고 플레이어 오브젝트의 z 좌표를 이용하여 플레이어의 현재 높이를 추정하고, 땅 속으로 꺼지는 문제를 방지하기 위하여 최솟값을 적용해준 뒤 플레이어 컨트롤러의 Havok 충돌체의 크기를 리사이징 해주었습니다.