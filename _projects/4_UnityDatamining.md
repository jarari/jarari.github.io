---
name: Unity Engine 데이터마이닝
tools: [C++]
image: assets/U_Logo_Small_Black_CMYK_1C-1589x537-ada00fa.png
description: 국내외 Unity Engine 기반 게임의 암호화 파일 탐구
---

# Unity Engine 데이터마이닝

저는 개인적인 취미로 제가 플레이하는 여러 유니티 엔진 기반 게임들의 클래스 구조, 구현 방식, 서버 통신, 에셋 번들 구조들을 살펴보고 분석해보았습니다.<br>
<br>
### 왜 Unity Engine 게임들을 살펴보았는가?
<br>
C#을 사용하는 유니티 엔진의 특성상 클래스 구조 및 글로벌 변수들이 외부에 그대로 드러나기 때문입니다. 대부분 실제 발매된 유니티 게임들의 경우에는 il2cpp를 사용하여 컴파일되어 있어 코드 소스까지 외부로 드러나있는 경우는 잘 없지만, il2cpp API를 사용하여 타입과 함수를 검색하고 변환해서 사용하기 때문에 특정 도구들과 리버스 엔지니어링을 통해 클래스명, 함수명, 타입, 상속구조, 글로벌 변수 등 구현 방식을 엿볼 수 있는 기회가 많습니다. 유니티 엔진을 사용하는 게임도 많고 해당 게임을 플레이하는 유저 풀도 커서 데이터마이닝에 필요한 도구들이나 정보가 빠르게 퍼진다는 점도 저에게는 장점이었습니다.<br>
## 게임들의 특징
<br>
### 에테르 게이저 (深空之眼)
![에테르 게이저](assets/skzy.png)
에테르 게이저는 벽람항로의 공동 개발사로 유명한 Yongshi의 2번째 작품이며, 모바일 및 PC에서 플레이 가능한 ARPG 게임입니다. 주요 구현 특징으로 모든 스텟 데이터, 대사, 번역, 추가 컨텐츠 등등이 Lua 스크립트를 통해 제공된다는 점입니다. global-metadata.dat의 암호화를 통해 il2cpp 분석을 어렵게 만들어놓은 점 또한 눈여겨 볼 부분이며, 에셋 번들에 대해서는 별도의 암호화 없이 파일 맨 앞에 공백 바이트(00)들을 추가하여 Asset Studio에서 읽을 수 없도록 만들어두었습니다. 버전마다 json 형태의 에셋 해시 데이터 파일을 이용하여 외부 리소스를 서버에서 다운받도록 하고 있으며, 해당 데이터 파일에는 번들의 원본 경로, 용량이 기록되어 있어 데이터를 읽고 변조를 확인할 때 사용합니다. 또한, 프록시 사용 시 혹은 허용되지 않은 인증서를 통해 통신 시도 시 연결을 거부하도록 되어있고 내부 API를 이용하여 버전 정보 등 여러 데이터를 받아오기 때문에 유니티 엔진을 사용한 것에 비해 높은 보안 수준을 자랑합니다. 다만, 공개 테스트 서버의 CDN이 유추하기 쉬운 형태로 되어있어 개발중인 리소스가 유출되기 쉽습니다.<br>
{% include elements/figure.html image="projects/assets/skzy_assethash.png" caption="에셋 해시 데이터" %}
<br>
오디오 및 영상 엔진도 보통 일본에서 자주 사용하는 CRIWARE을 사용하고 있으며, 64비트 암호로 보호되어 있어 쉽게 접근하기 어렵습니다. 다만, 해당 미들웨어 특성 상 암호키를 정의해야 하는데 이를 에셋 번들 내부의 MonoBehavior을 통해 하고 있어 암호화 키가 외부에 노출되어 있습니다. 이를 이용하면 원본 미디어를 복구할 수 있습니다.<br>
{% include elements/figure.html image="projects/assets/skzy_criware.png" caption="무방비로 노출된 암호화 키" %}
<br>
### 승리의 여신: 니케
![승리의 여신: 니케](assets/nikke.jpg)
승리의 여신: 니케는 데스티니 차일드와 김형태 CEO님으로 대표되는 SHIFT UP의 2번째 작품이며, 모바일 및 PC에서 플레이 가능한 미소녀 슈팅 게임입니다. SHIFT UP의 아트 및 음성 리소스에 대한 사랑은 정말 각별합니다. 니케들의 Spine 2D 데이터, SD 모델, 음성, 배경음악 등 각종 에셋들이 NKAB 라는 특수한 번들로 제공되며, 게임에서 해당 번들을 로드할 때 Addressables에 기록된 정보를 통해 암호화 번들인지 확인하고 복호화 한 후 게임에 로드됩니다. 특징으로는 파일 헤더에 NKAB 라는 매직 바이트와 해당 암호화 알고리즘의 버전 넘버가 적혀있고, 버전에 따라 파일 복호화에 필요한 각종 파라미터값, AES 키, AES IV, 솔트 등이 적혀있습니다. NKAB 버전 3부터는 유니티 엔진의 IKeyProvider를 이용하여 파일마다 다른 키를, 버전 당 4가지 정도 사용하고 있으며 NKAB 버전 4에서는 libsodium을 통한 파일 무결성 검증용 시그니쳐 64바이트가 추가되었습니다.<br>
{% include elements/figure.html image="projects/assets/nikke_nkab.png" caption="암호화된 NKAB v4 번들 (좌) 와 복호화된 번들 (우)" %}
{% include elements/figure.html image="projects/assets/nikke_nkab2.png" caption="IDA 분석을 통해 알아낸 복호화 과정을 C#으로 구현" %}
게임에 사용되는 니케, 무기, 몬스터의 스텟 및 게임 세팅, 전투력 등등의 설정은 json 파일 형태로 StaticData.pack이라는 파일에 들어있습니다. 클라이언트가 접속하는 서버에 특정 리퀘스트를 보내면 서버가 해당 파일을 다운받을 수 있는 주소와 솔트 2개를 제공하는데, 이 둘은 StaticData.pack의 이중 암호화를 해제하는데에 한 번씩 사용됩니다. 해당 파일은 AES-128과 AES-CTR로 암호화된 압축 파일이며 이 알고리즘은 처음 출시 이후 단 한 번만 업데이트 되었습니다.<br>
{% include elements/figure.html image="projects/assets/nikke_staticdata.png" caption="StaticData.pack은 이중으로 암호화된 zip파일" %}
최근에는 Addressables 변조를 통해 NKAB 암호화를 우회하여 변조된 파일을 게임에 로드시키는 제3자 프로그램을 막기 위해 Addressables 파일 및 로컬라이제이션 데이터들을 sqlite3 데이터베이스로 변환하고, NKDB 라는 새로운 암호화 포맷을 도입하였습니다. 해당 파일 또한 변조되는 것을 막기 위해 libsodium의 public-key signatures를 이용하여 검증하고 있습니다. 데이터 보호와 보안에 정말 진심이 아닐 수 없습니다.
{% include elements/figure.html image="projects/assets/nikke_nkdb.png" caption="복호화된 Addressables 데이터베이스" %}