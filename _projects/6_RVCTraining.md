---
name: RVC Training with Game Voice Assets
tools: [C++]
image: assets/rvc.png
description: Retrieval-based Voice Conversion을 이용한 게임 캐릭터의 음성 특징 학습과 변환
---

# RVC Training with Game Voice Assets

[Retrieval-based Voice Conversion (RVC)](https://github.com/RVC-Project/Retrieval-based-Voice-Conversion-WebUI)는 자체적으로 Pre-trained 된 모델과 2020년 카카오엔터프라이즈에서 발표한 [HiFi-GAN](https://arxiv.org/pdf/2010.05646) 모델을 이용하여 음성 데이터셋에서 특징점들을 추출 및 학습한 후 효과적으로 주어진 음성 데이터의 목소리를 학습된 발화자의 목소리처럼 느껴지는 음성 데이터로 변환하는 기술입니다. 대중에게는 흔히 "AI 아이유", "AI 김광석" 등 AI 커버곡들을 제작하는데에 사용하는 것으로 알려져 있습니다.<br>
<br>
해당 기술을 이용하기 위해서는 일단 음성 데이터셋을 학습시켜 발화자 모델을 제작해야 하는데, 저는 게임에서 사용되는 캐릭터 음성들은 대부분 효과가 적용되지 않은 스튜디오 퀄리티의 순수한 사람 목소리라는 점에서 착안하여, 게임에서 추출한 음성 데이터를 이용한 모델을 제작하고 이를 활용할 수 있는지 탐구하여 보았습니다.<br>

### Applio
![Applio](assets/rvc_applio.png)
[Applio](https://applio.org/)는 RVC를 조금 더 쉽게 사용할 수 있도록 여러가지 플러그인과 음정 추출 알고리즘을 추가한 프로젝트입니다. 저는 모든 학습과 추론을 Applio에서 제공하는 웹 기반 UI에서 진행하였습니다.

### Training
{% include elements/figure.html image="projects/assets/rvc_loss.png" caption="Generator Loss" %}
HiFi-GAN은 Generative Adversarial Network 모델을 이용하여 음성 데이터를 합성합니다. GAN은 데이터를 생성하는 Generator와 데이터를 구별하는 Discriminator를 서로 대립시켜(adversarial) Generator는 최대한 진짜같은 데이터를 만들어내고, Discriminator는 생성된 데이터와 학습 데이터로 이루어진 문제에서 이를 구분하게 만듭니다. 이러한 경쟁구도에서 Generator는 Discriminator를 속일 수 있도록 점점 사실적인 데이터를 만들어내게 되고, Discriminator는 Generator가 제공하는 가짜 데이터를 구분하기 위해 강화되게 됩니다. 이 때 중요한 것이 모델이 underfit이나 overfit되지 않도록 주의하는 것인데요, underfit은 말 그대로 학습이 덜 된 상태를 의미하고 overfit은 비유하자면 기출만 잘 푸는 학생같은 상태가 됨을 뜻합니다. 이를 가려내기 위해 tenserboard를 통해 Generator의 loss율을 관찰하였고, 이 값이 안정화되는 스텝의 모델들을 직접 테스트해보며 잘 만들어진 모델을 추려냈습니다.<br>
<br>
음정 추출 알고리즘 또한 중요한데, 여러 알고리즘을 테스트해본 결과, rmvpe가 평균적으로 좋은 결과를 보였고 학습 속도 또한 다른 알고리즘에 비해 빨랐습니다. crepe의 경우 학습 속도가 rmvpe에 비해 3~4배정도 오래 걸렸으나 몇몇 코너 케이스에서 더 정확한 결과를 보였습니다. 완벽한 결과를 위해서는 두 알고리즘을 사용한 모델을 모두 제작한 후 더 좋은 품질의 오디오를 합성하여 사용하는 것이 좋습니다.

### Results
<p class="text-center">
    <audio src="assets/rvc_fo4_original.wav" controls preload></audio><br>
    폴아웃 4 대사
</p>
학습된 모델들을 검증하기 위하여 폴아웃 4에서 추출한 여성 주인공의 대사를 변환시켜보았습니다.<br>
좌측은 데이터셋에 사용된 음성이며, 우측은 RVC를 통해 변환된 음성입니다. 품질 검증을 위하여 추론된 음성 데이터에는 어떠한 수정도 가하지 않았습니다.<br>

| |Original      | Converted
|-------|--------|---------|
Model 1    | <audio src="assets/rvc_model1_original.wav" controls preload></audio> | <audio src="assets/rvc_model1_converted.wav" controls preload></audio>
Model 2    | <audio src="assets/rvc_model2_original.wav" controls preload></audio> | <audio src="assets/rvc_model2_converted.wav" controls preload></audio>
Model 3     | <audio src="assets/rvc_model3_original.flac" controls preload></audio> | <audio src="assets/rvc_model3_converted.wav" controls preload></audio>
Model 4     | <audio src="assets/rvc_model4_original.flac" controls preload></audio> | <audio src="assets/rvc_model4_converted.wav" controls preload></audio>

### Conclusion
게임에 사용되는 것과 같이 높은 품질의 데이터셋을 이용하면 따로 후처리 과정 없이도 발화자를 구분하기 힘들 정도의 음성 데이터를 빠르게 생성할 수 있었습니다. 또한, Model 4를 통해 여성의 음성 데이터를 입력하고 남성의 모델을 사용하더라도 상당히 자연스러운 음성 데이터가 출력됨을 확인할 수 있었습니다. 이를 이용하면 게임 내에서 ChatGPT, TTS, 그리고 RVC를 결합하여 해당 캐릭터의 성우와 동일한 목소리로 실시간 대화가 가능하지 않을까 기대할 수 있는 부분입니다.