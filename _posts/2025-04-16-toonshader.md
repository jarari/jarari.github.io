---
title: >
    Unity Toon Shader
tags: [Unity, Shader]
style: border
color: primary
description: >
    유니티 공식 패키지 Unity Toon Shader 사용해보기
---

![콜펜](assets/toon.png)

아니메 스타일 모바일 게임들이 인기를 끌고 있기도 하고, 저도 개인적으로 그런 게임을 많이 플레이하고 있기에 평소 카툰 렌더링 방식에 대한 유튜브 영상을 시청해보거나 리소스들을 다뤄보고 있었는데요. 이번에 턴제 게임을 하나 제작해보면서 카툰 렌더링 캐릭터를 하나 넣어보고 싶어 쉐이더를 조금 공부해보다보니 [Unity Toon Shader](https://docs.unity3d.com/Packages/com.unity.toonshader@0.11/manual/index.html) 라는 패키지가 있어 한 번 사용해 보았습니다.

![패키지 매니저](assets/toon2.png)

툰 쉐이더는 패키지 매니저의 Add package by name 기능을 이용하여 설치할 수 있으며, `com.unity.toonshader` 를 입력하시면 바로 사용 가능한 쉐이더 파일과 필요한 리소스들이 다운로드 됩니다.

![마테리얼 설정](assets/toon3.png)

마테리얼 에셋의 Shader를 Toon으로 설정하면 바로 적용되게 되며, Mode에서 세부 맵들을 추가적으로 표시하는 모드로 변경할 수도 있으나 웬만해서는 일반 모드에서도 원하는 셸 쉐이딩 효과를 다 구현할 수 있을 것 같습니다.<br>
<br>
일단 제일 중요한 설정은 Base Map, 1st Shading Map, 2nd Shading Map인데요. 해당 쉐이더는 빛에 따라 3단계에 걸쳐 그림자를 표현해주는 방식이고, 가장 밝을때는 Base Map, 1차 쉐이딩은 1st Shading Map, 2차 쉐이딩은 2nd Shading Map을 이용하여 표현됩니다.

![2단계 그림자](assets/toon4.png)

저는 간단하게 Base Map과 1st Shading Map이 동일한 맵을 사용하도록 하고 1st Shading Map에 피부보다 조금 어두운 색상을 주는 것으로 2단계 그림자를 표현하였습니다.

![2단계 그림자 머리](assets/toon5.png)

머리카락의 경우에는 푸른 계열의 색을 주어 자연스러운 그림자가 지도록 구성하였습니다.

![Shading Step](assets/toon6.png)

Shading Step and Feather Settings 탭의 설정을 이용하면 각 맵이 그림자에서 차지하는 비율을 조절할 수도 있습니다. 위 사진은 각각 Base Color Step 에 0, 0.5, 1.0의 값을 준 모습입니다.<br>
Feather 설정 값들을 이용하면 그림자가 계단식이 아니라 경계면에서 그라데이션 형태로 번지는 느낌도 줄 수 있습니다.

![외곽선](assets/toon7.png)

간단한 외곽선 효과도 자체적으로 지원해줍니다. Outline을 체크하면 활성화되고, Outline Mode에서 Normal Direction 또는 Position Scaling 모드를 사용할 수 있으며 여러 세부 항목도 조절할 수 있습니다.<br>
모드에 따라 외곽선이 입혀지는 방식에 약간의 차이가 생기는데, 본질적으로 둘 다 해당 메쉬의 크기를 키운 후 Face의 앞면을 컬링하고 색을 덮는 방식으로 구현되어 있지만 Normal Direction의 경우 메쉬의 Face normal을 따라서 면을 띄우는 방식으로 렌더링되고 Position Scaling은 단순히 메쉬의 크기를 중심 기준으로 키워서 표현되기 때문에 상황에 맞게 적절히 설정하여 사용하면 될 것 같습니다.<br>
사실 어떤 방식을 써도 모델이 이 기능을 염두에 두고 만들어진게 아니라면 완벽하지는 않기 때문에 비용 절감 측면이 아니라면 포스트 프로세싱 단계에서 외곽선 검출 기법을 이용해 그려주는게 제일 낫지 않나 싶습니다.