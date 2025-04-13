---
title: >
    Unity ScriptableObject
tags: [Unity]
style: border
color: primary
description: >
    데이터 관리를 용이하게 해주는 ScriptableObject 탐구
---

바보같은 고민을 한 적이 있습니다. 스텟 데이터를 대체 어디에 담을까... 객체마다 다 다를 필요는 없는데... JSON으로 만들어서 로드해야하나? 그냥 Scriptable Object를 쓰면 되는거였는데요.<br>
<br>
스크립터블 오브젝트는 에셋의 형태로 저장하고 스크립트에서 참조할 수 있는 데이터 컨테이너입니다. 굳이 여러번 정의할 필요 없는 데이터를 참조하는데에 사용할 수도 있고, 인스펙터에 값들이 노출되다 보니 프로그래머가 알고리즘에 사용되는 여러 변수들을 스크립터블 오브젝트에서 불러오도록 설정만 해준다면 프로그래밍을 모르는 다른 사람들도 쉽게 수치를 변경하고 테스트 할 수 있어 협업에도 용이합니다.<br>

```C#
public enum CharacterType {
    Action,
    Shooter
}

[System.Serializable]
public struct CharacterStats {
    [field: SerializeField]
    public float CurrentHP { get; set; }
    [field: SerializeField]
    public float MaxHP { get; set; }
    [field: SerializeField]
    public float CurrentGroggy { get; set; }
    [field: SerializeField]
    public float MaxGroggy { get; set; }
    [field: SerializeField]
    public float CurrentSkillPts { get; set; }
    [field: SerializeField]
    public float UseSkillThreshold { get; set; }
    [field: SerializeField]
    public float MaxSkillPts { get; set; }
    [field: SerializeField]
    public float CurrentUltPts { get; set; }
    [field: SerializeField]
    public float Attack { get; set; }
    [field: SerializeField]
    public float Defense { get; set; }
    [field: SerializeField]
    public float CritChance { get; set; }
    [field: SerializeField]
    public float CritMult { get; set; }
    [field: SerializeField]
    public float DefFlatPenetration { get; set; }
    [field: SerializeField]
    public float DefPercentagePenetration { get; set; }
}

[CreateAssetMenu(fileName = "Data", menuName = "ScriptableObjects/CharacterData", order = 1)]
public class CharacterData : ScriptableObject {
    public CharacterStats stats;
    public CharacterType charType;
}
```

저는 이런 형태의 캐릭터 데이터를 한 번 만들어 Character 클래스에 베이스 데이터로 등록할 수 있게 하고, CurrentHP와 CurrentGroggy 등 변형이 필요한 변수들이 있기 때문에 Awake에서 Instantiate 해주는 방식으로 런타임용 데이터를 따로 생성해서 사용하였습니다.<br>
유니티 게임은 데이터와 기능을 최대한 분리, 모듈화하여 관리하는 것이 좋은 것 같다는 것을 다시 한 번 느꼈습니다.