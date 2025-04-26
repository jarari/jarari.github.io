---
title: >
    Unity Reflection Probe
tags: [Unity]
style: border
color: primary
description: >
    Reflection Probe를 이용한 환경 반사 세팅
---

![Reflection Probe](assets/reflection.png)

유니티 URP 환경에서 환경 반사를 지원하는 마테리얼들은 Lighting 탭의 Environment -> Environment Reflections 설정에 따라 표면의 반사가 일어나게 됩니다. 기본 설정은 Skybox고, 반사 강도나 재질간 최대 반사 횟수는 1로 설정되어 있습니다.

![푸르스름한 텍스처](assets/reflection2.png)

이때 흔히 발생할 수 있는 문제가 텍스처에 나타나는 푸르스름한 빛인데요, 이는 Default-Skybox가 반사된 큐브맵이 이렇게 새파란 색이기 때문입니다.<br>
별도의 조치를 해주지 않으면 해당 모델이 월드의 어디에 위치해있든 스카이박스를 이용하여 베이킹한 큐브맵 혹은 커스텀으로 설정한 큐브맵만이 사용되게 되는데, 이 때 사용할 수 있는 것이 Reflection Probe 입니다.

### 하지만 일단 Cubemap부터 알아보자

{% include elements/figure.html image="blog/assets/reflection3.png" caption="출처: https://math.hws.edu/eck/cs424/graphicsbook-1.3/c5/s3.html" %}

큐브맵은 이름 그대로 정육면체로 구성된 맵입니다. 유니티에서는 파노라마 사진을 하나 임포트 한 후, Texture Shape를 Cube로 지정하고 Skybox/Cubemap 쉐이더를 사용하는 마테리얼을 생성하면 쉽게 제작할 수 있습니다.<br>
이렇게 만들어진 맵은 빛을 반사하는 특성을 가진 재질이 어떤 색을 가져야하는지 빠르게 연산하는데에 쓰입니다. 요즘은 레이트레이싱 기술을 기반으로 실시간 반사를 계산할 수도 있지만, 이러한 기능은 막대한 연산 능력이 필요하기 때문에 큐브맵 반사가 여전히 유용하게 쓰이고 있습니다.

![큐브맵의 원리](assets/reflection4.png)

색상을 가져오는 원리를 간단하게 나타나면 위와 같습니다. 빨간색 선이 빛이라고 가정하였을 때, 검은 구에 반사된 빛이 큐브맵과 교차하는 지점인 초록 X표 지점의 텍스처 픽셀값을 이용하여 색상을 블렌딩해주는 것이죠.

### Reflection Probe

![세부 설정](assets/reflection5.png)

리플렉션 프로브는 이런 큐브맵을 현재 씬을 기반으로 바로 제작할 수 있게 도와주고, 환경 반사에 사용할 수 있게 해줍니다.<br>
타입 설정의 Baked나 Custom을 이용하면 미리 큐브맵을 제작해두고 사용할 수 있습니다. Baked를 사용하면 큐브맵은 씬 자체의 파일로 저장되게 되며, Custom을 사용하면 큐브맵을 지정하여 사용하거나 현재 씬을 기반으로 큐브맵을 만들어서 exr 파일로 보관할 수 있습니다.<br>
Realtime 타입은 오브젝트가 활성화 될 때, 혹은 매 프레임마다 반사 효과가 업데이트되게 할 수 있으나 이를 사용하기 위해서는 Project Settings -> Quality에서 Realtime Reflection Probes 옵션을 활성화시켜줘야 합니다.

![범위](assets/reflection6.png)

리플렉션 프로브는 적용 범위도 설정할 수 있습니다. 해당 범위 내에서 렌더링되는 메쉬들의 환경반사에만 리플렉션 프로브가 사용되게 되며, 밖의 범위에는 기본 설정이 사용됩니다.<br>
두 개 이상의 리플렉션 프로브가 씬에 존재하는 경우 더 높은 우선순위를 가진 리플렉션 프로브가 사용됩니다. 그리고 조명 설정에 따라 여러 리플렉션 프로브가 블렌딩되기도 합니다.

![베이크](assets/reflection7.png)

큐브맵을 생성하려면 Bake 버튼을 눌러주기만 하면 됩니다. 이때 생성되는 큐브맵은 단순히 리플렉션 프로브를 기준으로 360도 카메라가 돌아간다고 생각하시면 편한데, 유니티 카메라와 비슷하게 해상도나 그림자 거리, 클리어 플래그 (비어있는 픽셀은 무엇이 렌더링 될 것인지), 컬링 마스크, 클리핑 플레인 설정으로 렌더링 될 물체와 아닐 물체를 정해줄 수 있습니다.

![스태틱 플래그](assets/reflection8.png)

큐브맵을 생성할 때 카메라 설정 이외에도 큐브맵에 렌더링 될 물체에는 Reflection Probe Static 플래그를 설정해주어야 합니다.<br>
이는 월드 오브젝트가 큐브맵 생성 이후에 움직이지 않을 것이다 라고 엔진에 알려주는 것과 비슷한 플래그라고 합니다.

![프리팹에 넣기](assets/reflection9.png)

Custom 타입으로 큐브맵을 생성하여 지정한 경우 프리팹에 리플렉션 프로브를 넣어둘 수도 있기 때문에 절차적으로 월드를 생성하는 경우 유용하게 사용할 수 있을 것 같습니다.