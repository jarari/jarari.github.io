---
title: >
    Unity URP Lit 기반 커스텀 쉐이더
tags: [Unity, Shader]
style: border
color: primary
description: >
    리소스에 맞게 쉐이더를 변형하여 사용해보기
---

![단각수](assets/customshader.png)

제작중인 턴제 게임에 사용할 적 리소스 중 _rmo 라는 특이한 타입의 텍스처를 사용하는 모델이 있어서 URP Lit 쉐이더를 기반으로 커스텀 쉐이더를 만들어보았습니다.

![에셋](assets/customshader2.png)

![블렌더](assets/customshader3.png)

조금 찾아보니 언리얼 엔진에서 사용하는 roughness, metallic, ao의 3 in 1 텍스처라고 하는데, 실제로도 그런 식으로 작동하는지 확인하기 위하여 블렌더에 같은 계열의 모델을 불러오고 쉐이더 노드를 이용하여 맵을 지정해보니 맞는 것 같았습니다.<br>
URP Lit 쉐이더를 기반으로 매핑만 수정해주면 비슷한 비주얼이 나올 것 같아 바로 URP에서 쉐이더와 에디터 관련 코드를 복사하고, 제가 사용할 부분들을 수정해 주었습니다.

![쉐이더 코드](assets/customshader4.png)
![쉐이더 코드2](assets/customshader5.png)
![쉐이더 코드3](assets/customshader6.png)

코드를 수정하던 도중 에셋 중에 비슷하면서 조금 다른 _smo 혹은 _smoe 맵을 사용하는 에셋도 있었는데, 블렌더에서 확인해보니 smoothness, metallic, ao, emission이 RGBA에 담긴 맵으로 보여 해당 에셋에도 대응할 수 있도록 쉐이더 코드를 수정해주었습니다.<br>
마테리얼을 지정하는 디자이너가 rmo 혹은 smo 워크플로우를 선택할 수 있게 하고, 해당 워크플로우에 맞게 rmo는 smoothness 에 1 - R채널 * Smoothness, metallic에 G채널 * Metallic, occlussion에 B채널 * Occlusion Strength, emission이 체크된 경우 emission에 A채널 * Emission Color를 넣어주고, smo는 smoothness에 R채널 * Smoothness을 넣어주도록 하였습니다.

![에디터 코드](assets/customshader7.png)
![인스펙터](assets/customshader8.png)

에디터 코드 또한 이에 맞게 수정하여 인스펙터에서 적절한 옵션들이 표시될 수 있도록 변경하였습니다.<br>

![외곽선](assets/customshader9.png)

외곽선이 그려지면 좋을 것 같아 오브젝트를 Face normal 방향으로 늘려주고 Front를 컬링하는 쉐이더 코드도 작성하였습니다.

![옅어지는 외곽선](assets/customshader10.png)

외곽선은 카메라로부터 Outline Fade Start 거리만큼 떨어지면 페이드가 시작되어 Outline Fade End 거리에서 알파값이 0이 되도록 하였습니다.