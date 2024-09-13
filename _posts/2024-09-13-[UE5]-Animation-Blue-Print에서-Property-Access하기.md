---
title: "[UE5] Animation Blueprint의 Thread-UnSafe Warning 해결하기"
author: seolwon
date: 2024-09-13 16:00:00 +0900
media_subpath: \assets\img\posts\2024-09-13-[UE5]-Animation-Blue-Print에서-Property-Access하기
categories: [UnrealEngine5, Blueprint]
tags: [UnrealEngine5, UE5, AnimationBlueprint, Thread-Safe, Warning]
---

`Animation Blueprint`를 사용하면서, 다른 블루프린트의 변수에 접근할 때 이런 오류가 뜬적이 있을 것이다.
```md
Node  Blendspace Player 'BS_Sword_Character_Move'  uses potentially thread-unsafe call  Get Player Controller . 
Disable threaded update or use a thread-safe call. Function may need BlueprintThreadSafe metadata adding. 
```

위의 오류는 `BS_Sword_Character_Move`인 블렌드 스페이스는 잠재적으로 스레드에 안전하지 않은 호출인 `Get Player Controller`를 사용하고 있음으로, 스레드에 안전한 호출을 사용하라는 뜻이다.<br>
그렇다면 이런 오류는 왜 뜨는 것일까?<br>

여러분이 `Animation Blueprint`를 사용해 보았다면, `Animation Blueprint`는 다른 `Blueprint`가 통상적으로 가지고 있는, `Event Graph`뿐만이 아니라 `Anim Graph`또한 가지고 있는 것을 알것이다.<br>
**바로 여기서 문제가 생기는데, 기본적으로 AnimGraph는 EventGraph와 별도의 CPU로직에서 실행된다.**
`Animation Blueprint`와 다른 모든 `Blueprint`가 실행되는 CPU 스레드를 `Game Thread`라고 하고, 이와 별도로 실행되는 `Anim Graph`가 실행되는 쓰레드를 `Worker Thread`라고 한다.
요는 결국 두 `Graph`는 다른 스레드에서 실행되기 때문에, **다른 스레드에서 실행된 `Get Player Controll`를 사용해서 변수를 가져오려고 하고 있기때문에, Thread-Unsafe Warning**이 발생하는 것이다.<br>

그렇다면 이걸 어떻게 해결해야 할까?<br>
바로 **Property Access Node**를 사용해서 해결할 수 있다.<br>
밑의 이미지를 보자<br>
![Img-01](/09-13_Img1.png)
<br>이렇게 **Property Access Node**를 만들려면 `Anim Graph`에서 마우스 오른쪽 버튼을 클릭하고 `Variables`에서 `Property Access`를 선택하면 된다.<br>

추가한 후에는 노드에서 드롭다운 메뉴를 열고 원하는 `Get` 함수를 클릭하면 된다. 단일 `Get`에서 타고 타고 넘어가서 정말로 원하는 구체적인 `Property`를 찾을 수도 있다.<br>
![Img-02](/09-13_Img2.png)
<br>이렇게 바인딩한 `Property Access Node`를 가지고 그래프에 변수를 제공할 수 있다.<br>
![Img-03](/09-13_Img3.png)