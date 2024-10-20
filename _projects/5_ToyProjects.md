---
name: Toy Projects
tools: [C++]
image: assets/grapplinghook.png
description: 가벼운 프로젝트 모음
---

# Toy Projects

따로 페이지를 할애하기엔 이야기할게 많지 않지만, 충분히 흥미로운 프로젝트 모음입니다.<br>
<br>
### Grappling Hook
{% include elements/video.html id="dncTKnTlbOA" %}
폴아웃 4에 있는 Arrow Projectile (지형지물에 부착되는 프로젝타일) 과 Bendable Spline (로프를 표현하는데에 사용되는 오브젝트)을 사용하여 발사하고 부착되는 갈고리를 구현하고, 실시간으로 액터 가속도를 수정하여 에이펙스 레전드 스타일의 그래플링 훅 시스템을 구현한 모드입니다. 프레임 레이트에 따른 오차가 최소화되도록 틱레이트를 지정하여 어떤 상황에서든 비슷한 움직임이 나타나도록 구현하는데에 집중하였으며, 세이브 파일 시리얼라이즈 인터페이스를 이용하여 세이브 파일 간에서도 그래플링 훅 상태가 유지되도록 하였습니다.
<p class="text-center">
{% include elements/button.html link="https://github.com/jarari/GrapplingHook" text="프로젝트 GitHub" %}
</p>
### Simple Impact
{% include elements/video.html id="_s7pyNSw45U" %}
폴아웃 4의 건파이트 타격감을 증가시키기 위하여 발사 시 카메라 쉐이크, 블러 및 이미지스페이스 오버레이를 추가하고 피격 부위에 따른 히트마크 사운드를 추가하는 모드입니다. 유저가 별다른 설정을 하지 않아도 적절한 강도값이 사용되도록 무기의 데미지, 발사 속도 등을 고려해 점수를 매기는 휴리스틱 함수를 구현하였습니다.
<p class="text-center">
{% include elements/button.html link="https://github.com/jarari/SimpleImpact" text="프로젝트 GitHub" %}
</p>
### Auto Beam
{% include elements/video.html id="WN_E_tuR1CA" %}
폴아웃 4의 무기 부착물 중 장식에 불과한 레이저를 Havok 레이캐스트와 수학의 힘으로 플레이어가 조준하는 곳에 정확히 위치하도록 보정해주는 모드입니다. 실제 총구 방향과 플레이어의 조준 방향도 일치하지 않는 경우가 대부분이었기 때문에 현재 플레이어의 상태, 총구와 조준점의 각도 차이 등 여러 조건을 고려하여 보정 여부를 결정하였고, 총구 움직임을 어느 정도 따라가도록 보정값들을 일정 크기로 샘플링하여 낸 평균값을 이용하였습니다.
<p class="text-center">
{% include elements/button.html link="https://github.com/jarari/AutoBeam" text="프로젝트 GitHub" %}
</p>