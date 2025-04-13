---
title: >
    Unity ObjectPool
tags: [Unity]
style: border
color: primary
description: >
    유니티 2021부터 자체 제공되는 오브젝트 풀링 체험
---

이쯤되면 누구나 다 알듯, 반복적인 Instantiate 와 Destroy는 성능 측면에서 안좋습니다. 첫번째로 오브젝트를 소환할 때 메모리를 할당하고 오브젝트가 생성되었음을 처리해야하니 CPU 타임이 소모되고, 두번째로는 Destroy를 하면 메모리가 해제되었으니 Garbage Collector가 이를 처리해야하는데 GC는 게임을 멈추고 (stop-the-world) 분리수거를 진행하기 때문에 이 작업이 끝날 때 까지 CPU 스파이크와 프레임 저하가 일어납니다. 유니티에서 점진적 가비지 컬렉션 (Incremental garbage collection)을 사용하고 있다곤 하나, 근본적으로 GC를 빠르게 만드는 것은 아니고 GC 작업을 여러번으로 나누어 할 뿐 총량이 늘어나면 성능에 해가 되는 것은 매한가지 입니다.

![프로파일러](assets/pooling.png)

실제로 코드에서 총알에 오브젝트 풀링만 빼도 에디터에서 250프레임 나오던 씬이 200프레임으로 떨어지고, 심지어 프로파일러상에서 GC가 호출되는 순간에는 140프레임까지도 곤두박질 칩니다.<br>
다행히도 유니티 2021.1부터 유니티에서 자체적으로 ObjectPool\<T0\> 이라는 클래스를 제공해주니, 이걸 사용해봅시다.

```C#
using UnityEngine;
using UnityEngine.Pool;

public class BulletManager : MonoBehaviour {
    public static BulletManager instance;
    public int maxPoolSize = 1000;
    public GameObject bulletPrefab;

    IObjectPool<Bullet> _pool;
    public IObjectPool<Bullet> Pool {
        get {
            if (_pool == null) {
                _pool = new ObjectPool<Bullet>(CreatePooledItem, OnTakeFromPool, OnReturnedToPool, OnDestroyPoolObject, true, 10, maxPoolSize);
            }
            return _pool;
        }
    }
    private void Awake() {
        if (instance != null) {
            Destroy(this);
            return;
        }
        instance = this;
    }

    private Bullet CreatePooledItem() {
        var go = Instantiate(bulletPrefab);
        return go.GetComponent<Bullet>();
    }

    private void OnReturnedToPool(Bullet bullet) {
        bullet.gameObject.SetActive(false);
    }

    private void OnTakeFromPool(Bullet bullet) {
        bullet.gameObject.SetActive(true);
    }

    private void OnDestroyPoolObject(Bullet bullet) {
        Destroy(bullet.gameObject);
    }

    public void SpawnBullet(Character attacker, Vector3 position, Vector3 dir, float speed = 50f, bool gravity = false) {
        Bullet b = Pool.Get();
        b.transform.position = position;
        b.transform.forward = dir;
        b.Initialize(attacker, speed, gravity);
    }
}
```

간단하게 총알을 소환하고 필요없어지면 꺼버렸다가 필요하면 다시 켜주는 최대 사이즈가 1000인 오브젝트 풀이 구현되었습니다. 단순히 new ObjectPool\<클래스\>만 입력하고, 처음 생성할 때 함수, 풀로 돌려보낼 때 함수, 풀에서 꺼내는 함수, 최종적으로 Destroy하는 함수까지만 구현하면 끝이라니, 정말 좋네요.

```C#
    private void ReturnToPool() {
        _hasCollided = true;
        _rigidbody.isKinematic = true;
        BulletManager.instance.Pool.Release(this);
    }
```

Bullet 클래스에는 간단하게 풀로 되돌려주는 함수만 구현해주면 되겠습니다.