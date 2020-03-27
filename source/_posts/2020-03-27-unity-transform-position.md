---
title: transform.position
date: 2020-03-27 21:56:54
tags: ["unity", "C#", "position", "axis", "deltatime"]
categories: ["Unity", "C#"]
comments: true
---

# Intro

유니티 UI에서 `Hierarchy` 탭에 존재하는 오브젝트를 클릭하면, `Inspector` 탭에 해당 오브젝트의 컴포넌트들이 표시된다.

그중에 오늘은 __Transform__, 그중에 `Position`에 대해 간략하게 메모를 해본다.


# Transform

`Transform` 컴포넌트 안에는 `Position`, `Rotation`, `Scale` 3가지 필드가 존재하며 각각 위치, 회전, 크기를 담당한다.

![Unity - Transform UI](/images/unity_transform.png)
 
재밌는건 위의 UI의 값들을 C# 스크립트 내에서도 사용이 가능하다.

먼저 `Position` 관련 값들을 스크립트로 조작하기 위해서는 `Global Axis`와 `Local Axis` 개념을 알아야 한다.

간단하다.

`Global Axis`는 배경(?) 세계(?) 기준의 좌표계이고,
`Local Axis`는 선택된 게임 오브젝트 기준의 좌표계이다.

먼저 `Global Axis`의 경우 `Position`을 이동하는 스크립트를 짜볼껀데,
🆆(앞쪽), 🅰(왼쪽), 🆂(뒤쪽), 🅳(오른쪽) 단축키로 이동한다고 가정하도록 해보자.

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    void Start() {}

    void Update()
    {
        if (Input.GetKey(KeyCode.W))  // 🆆 누를 경우
            transform.position += new Vector3(0.0f, 0.0f, 1.0f);

        if (Input.GetKey(KeyCode.S))  // 🆂 누를 경우
            transform.position += new Vector3(0.0f, 0.0f, -1.0f);

        if (Input.GetKey(KeyCode.A))  // 🅰 누를 경우
            transform.position += new Vector3(-1.0f, 0.0f, 0.0f);

        if (Input.GetKey(KeyCode.D))  // 🅳 누를 경우
            transform.position += new Vector3(1.0f, 0.0f, 0.0f);
    }
}
```

위 코드는 `Global Axis` 기준으로 `new Vector3(float x, float y, float z)` 를 이용해 `transform.positon`의 값을 증가시키는 스크립트다.

🆆를 누르면 z 좌표를 1씩 증가시켜 앞으로 전진하고, 🆂를 누르면 z좌표를 -1씩 증가시켜서 뒤로 이동하고, 🅰와 🅳는 x좌표를 수정해서 좌우로 이동한다.

여기에서 `new Vector3()` 부분이 사실 좀 귀찮은데, Unity는 친절하게도 자주 사용하는 기능들을 미리 만들어논게 많다. 문서를 잘 찾아보길 바란다.
아무튼 `new Vector3(0.0f, 0.0f, 1.0f)`는 `Vector3.forward`, `new Vector3(0.0f, 0.0f, -1.0f)`는 `Vector3.back`으로 바꿀 수 있고, 좌우도 `Vector3.left`, `Vector3.right`로 바꿀 수 있다. 


그럼 다시, `Global Axis`와 `Local Axis`는 무슨 차이가 있을까?
이를 알아보기 전에 UI에서 `Rotation`중 y값을 아무렇게나 수정한 뒤 위의 스크립트를 먼저 실행해보자. 어떤가?

게임 오브젝트를 y축으로 살짝 회전시키면 게임 오브젝트가 바라보는 방향이 아닌 항상 똑같은 방향으로 움직일 것이다. 왜냐하면 `Global Axis`기준으로 `Position`이 변경되기 때문이다. 굉장히 어색할 것이다.

그럼 이제 아래 스크립트를 작성해서 게임을 실행해보자. 물론 이전에 수정했던 `Rotation` 값은 그대로 두고 말이다.

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    void Start() {}

    void Update()
    {
        if (Input.GetKey(KeyCode.W))
            transform.Translate(Vector3.forward);

        if (Input.GetKey(KeyCode.S))
            transform.Translate(Vector3.back);

        if (Input.GetKey(KeyCode.A))
            transform.Translate(Vector3.left);

        if (Input.GetKey(KeyCode.D))
            transform.Translate(Vector3.right);
    }
}
```

`transform.Translate()` 함수는 친절하게도 `Local Axis`로 변환을 해준다.
따라서 게임 오브젝트가 y측으로 회전을 한 상태에서 🆆 키를 누르면 회전 후 바라보고 있는 방향을 앞으로 판단하고 움직이게 된다. 

# Time.deltaTime

마지막으로, 위의 스크립트들을 돌려보면 게임 오브젝트가 굉장히 빠르게 후다닥 움직일 것이다.
왜냐하면 `MonoBehaviour.Update()` 함수는 매 프레임마다 호출되고, 그러면 1초에 여러 프레임이 생성되면서 생성된 만큼 이동하기 때문이다.

자 그럼 이부분을 해결하기 위해서는?
학교에서 옛날에 물리를 배울때를 떠올려보자. 안배웠다면 어쩔 수 없지만...
`거리 = 속력 * 시간` 이라는 공식이 있다. 즉, 거리는 속력과 시간을 곱하면 구할 수 있다.
이게 갑자기 왜 나왔냐?

우리가 이동시키려는 1은 실제로는 거리이다. 1만큼 이동시키고 싶은데 `MonoBehavior.Update()` 함수는 프레임 단위로 수행되기 때문에 30frame/s일 경우 1초에 30번 호출하게 된다. 즉, 30frame/s의 속도로 이동한다.

그럼 `1 = 30 * t` 를 만족하는 시간 `t`를 우리가 알 수만 있다면, 프레임이 어쨌던 말든 우리는 원하는데로 1씩 이동시킬 수 있을 것이다.

이때 `t`가 바로 `Time.deltaTime`이다. 이 값은 지난 프레임이 완료되는 데 까지 걸린 시간을 나타내며, 단위는 초를 사용한다. 즉, 30프레임이라면 `Time.deltaTime`은 1/30s가 되는것이다.
그럼 `Time.deltaTime`을 위 스크립트에 적용해서 수행해보자.

```c#
using System.Collections;
using System.Collections.Generic;
using UnityEngine;

public class PlayerController : MonoBehaviour
{
    void Start() {}

    void Update()
    {
        if (Input.GetKey(KeyCode.W))
            transform.Translate(Vector3.forward) * Time.deltaTime;

        if (Input.GetKey(KeyCode.S))
            transform.Translate(Vector3.back) * Time.deltaTime;

        if (Input.GetKey(KeyCode.A))
            transform.Translate(Vector3.left) * Time.deltaTime;

        if (Input.GetKey(KeyCode.D))
            transform.Translate(Vector3.right) * Time.deltaTime;
    }
}
```

간단하게 `Time.deltaTime`을 곱해주면 된다.
한번 실행해보자.

너무 느리다면 상수를 더 곱해주면 된다. 거리는 속도와 시간과 비례하는 관계이니깐. :)