---
layout: single
title: 객체지향 프로그래밍의 복잡성을 절차지향으로 관리하기
date: 2023-05-07 21:30:00 +0900
categories: software
---

이 글에서는 객체지향 프로그래밍과 그와 동반되는 개념인 캡슐화가 어떤 방식으로, 왜 코드에 과도한 부수적 복잡성을 가져오는지, 그리고 이런 부수적 복잡성을 올바르게 관리하기 위해 절차지향 프로그래밍 방법을 올바르게 사용하는 방법에 대해 다루고자 합니다.

이 글은 Brian Will의 영상인 [Object-Oriented Programming is Bad](https://www.youtube.com/watch?v=QM1iUe6IofM) 에서 많은 내용을 발췌했고, 해당 영상에서 중요하다고 느꼈던 부분을 제가 이해한 방식대로 풀어내 보았습니다. 내용에 대해 더 관심이 있으시거나, 중간중간 이해하기 어려운 부분이 있으셨다면, 해당 영상도 같이 확인해보시길 추천드립니다.

## 객체지향과 캡슐화의 어려움

객체지향 프로그래밍을 정의하는 가장 중요한 요소 중 하나는 “캡슐화”(encapsulation)이다. 캡슐화를 간단히 말하면, 각 객체에게 공개되지 않는 상태(private state)가 있고, 다른 객체가 이 공개되지 않은 상태를 수정하러면 해당 객체의 메소드를 통해서 미리 해당 객체가 정해둔 방식으로만 변경할 수 있는 개념이다. 캡슐화 개념은 단일 객체를 이해하기 더 쉽게 만들 수 있지만, 여러 객체들 간의 의존성을 다루기 시작하면 캡슐화는 종종 역효과를 가져오기도 한다. 아래 문단에서는 이 내용에 대해 더 자세히 설명하고자 한다.

*(객체의 메소드를 호출하는 것은, 조금 더 추상적으로 표현하자면 객체에 메세지를 보낸다고 할 수 있다. 원 영상에서도 “메세지" 단어를 주로 사용하기 때문에, 아래 문단에서도 “메소드 호출” 과 “메세지 전송”을 같은 뜻으로 보고 혼용해서 사용하고자 한다.)*

### 다중 의존성과 전역변수화 문제

여러 개의 서로 다른 객체에서 한 개의 캡슐화된 객체로 메세지를 보낼 수 있게 하는 것 자체가 캡슐화를 유지하기 어렵게 만든다. 이는 객체에 메세지를 보내는 것은 곧 해당 객체의 상태를 읽고 변경할 수 있음을 의미하기 때문이다. 

예를 들어서 설명하자면, 아래 그림과 같이 B와 C, 두 객체가 A에게 메세지를 보내는 상황을 생각해 보자. C가 A의 상태에 어떻게 영향을 줄 수 있는지에 대한 정보를 B에서 알고 있어야만, B에서도 이에 맞춰 A에 알맞은 메세지를 보내서,  A의 상태를 올바르게 읽고 변경할 수 있을 것이다. 물론 객체 C의 입장에서 그 역 또한 성립한다 - C 또한 B를 알고 있어야 한다. 이런 다중 의존성이 많아진다면 A는 사실상 전역변수인것과 다름이 없어진다. 이처럼 의존하는 객체들끼리는 모두 간접적으로라도 서로를 알아야 하기 때문에, 자연스레 클래스의 경계가 모호해지고, 캡슐화의 의미가 크게 퇴색되는 것이다.

![[20:05](https://youtu.be/QM1iUe6IofM?t=1205)](/assets/images/2023-05-07/Untitled.png)
_[20:05](https://youtu.be/QM1iUe6IofM?t=1205)_
{: style="color:gray; font-size: 80%; text-align: center;"}

### 트리 의존성과 흩어진 관심사 문제

위와 같은 사실을 고려했을 때, 캡슐화된 객체의 의존성을 가장 적절히 관리하는 방법은 트리 형태를 취하는 것이다. 이 형태에서는 최상위 객체 하나를 제외하고는 모두 자신의 부모가 있고, 부모가 자식에게 메세지를 보내는 방식으로 객체들 간에 소통하게 된다. 이 방식은 클래스간의 관계가 잘 정돈되어 있는 것처럼 보이지만, 실제로는 흩어진 관심사 (cross-cutting concerns) 에 관련된 문제를 고려해야 한다. 최상위 객체에서 서로 다른 방향으로 깊숙히 내려간 객체 A와 객체 B가 서로 소통해야 하는 유스케이스가 발생한다면 어떨까? A와 B가 엄격하게 트리 구조를 따라가 상태를 바꾸는 것은 너무 복잡할 수 있다. A와 B가 비슷하게 위치할 수 있도록 리팩토링하는 것도 방법이지만, 이는 즉 클래스간의 의존 구조를 완전히 바꿔야 하는 큰 변경을 의미한다. 이렇게 올바른 트리 구조를 무시하고 A와 B가 직접 소통한다면, 그러면 우리는 다시 위의 다중 의존성 문제로 돌아가게 된다. 그게 정말 최선이라면, 결국 이 캡슐화와 객체지향의 규칙이 우리에게 가치를 주고 있는 게 맞는가?

![[21:52](https://youtu.be/QM1iUe6IofM?t=1312)](/assets/images/2023-05-07/Untitled%201.png)
[21:52](https://youtu.be/QM1iUe6IofM?t=1312)
{: style="color:gray; font-size: 80%; text-align: center;"}

## 객체지향 프로그래밍의 본질

위에서 설명한 이런 객체 간 의존성 관리의 복잡성은 객체지향 프로그래밍의 어떤 특성에서 유도되는 것일까? 두 가지 객체지향 프로그래밍의 본질과 관련된 이유를 짚어볼 수 있다.

### 데이터와 함수의 이른 결합

객체지향 프로그래밍은 데이터와 함수를 묶는 것을 목표로 한다. 이 말인 즉슨, 우리의 코드는 본질적으로 여러 개의 데이터 타입과 행위(함수)로 구성되는데, 객체지향 프로그래밍이 추구하는 바는 모든 함수가 무조건 하나의 데이터 타입에 속하도록 코드를 구성하는 것이다. 

![[27:22](https://youtu.be/QM1iUe6IofM?t=1642)](/assets/images/2023-05-07/Untitled%202.png)
[27:22](https://youtu.be/QM1iUe6IofM?t=1642)
{: style="color:gray; font-size: 80%; text-align: center; width: 80%; margin: auto;"}

이런 이상향이 도움이 되는 상황은 명확히 존재한다. 그 예시로는 Queue, Stack과 같은 ADT(Abstract Data Type)를 생각해볼 수 있다. 이런 ADT는 데이터와 해당 데이터에 연결되어야 할 함수가 매우 명확하기 때문에, 클래스의 형태로 데이터와 함수를 연결하는 것이 매우 자연스럽고 유지보수성을 향상시킬 수 있다.

하지만 ADT와 같이 유용하고 깔끔한 추상화는 쉽게 만들어지지 않고, 긴 시간과 많은 고민을 필요로 한다. 이는 즉 우리가 비즈니스에서 주어진 데이터와 함수를 성급하게 클래스 단위로 묶었을 때, 그러한 묶음이 실제로는 가장 좋은 묶음이 아닐 가능성이 매우 크다는 점을 시사한다. 코드가 발전해가면서 원래의 데이터 타입 - 함수 매칭이 잘못된 것을 알게 될 수도 있는데, 그 매칭을 해체하고 새로 만드는 비용은 크다. 비유를 해 보자면, 우리가 집을 설계한 후 설계도대로 벽을 만들어 두었는데, 알고 보니 벽으로 분리된 방들끼리 물건을 주고 받아야 한다면, 우리에게는 벽을 따라 먼 길을 돌아가거나, 벽 사이에 개구멍을 내거나, 아예 벽을 허물어버리는 등의 썩 좋아 보이지 않는 선택지만 주어진다.

![[25:41](https://youtu.be/QM1iUe6IofM?t=1541)](/assets/images/2023-05-07/Untitled%203.png)
[25:41](https://youtu.be/QM1iUe6IofM?t=1541)
{: style="color:gray; font-size: 80%; text-align: center;"}

### 객체 단위의 이른 코드 분리

객체지향 프로그래밍의 또 다른 특징은 하나의 큰 코드를 여러 객체로 만들어서 서로간에 메세지를 전달하도록 하는 것을 장려한다는 것이다. 이렇게 하는 것은 분명 각각의 클래스의 범위를 줄여서, 각 클래스의 코드를 읽을 때의 인지적 부하를 줄여주는 역할을 한다. 하지만 코드를 분리하는 행위 자체는 코드 내의 비즈니스 로직의 본질적인 복잡도를 줄여주지는 않는다. 코드를 잘게 쪼개면 단지 원래는 한 파일에 있었던 비즈니스 로직이 서로 다른 파일, 서로 다른 클래스에 위치하게 되는 것이다. 로직들을 잘못 분리했을 때는, 원래 의도와는 정반대로 분리된 작은 조각간의 연결부위가 비대해져, 이를 이해하는 데 필요한 인지적 부하가 과도하게 커질 수 있다.

이는 마치 큰 큐브 하나를 작은 큐브로 나누는 것에 비유할 수 있다. 큐브를 나눈다고 그 부피는 줄어들지 않지만, 표면적은 크게 늘어난다. 충분히 많은 것을 고려하지 않고 큐브를 쪼갠다면 그 방대해진 표면적에서 오는 추가적인 복잡도를 같이 감당해야 할 것이다.

![[31:52](https://youtu.be/QM1iUe6IofM?t=1912)](/assets/images/2023-05-07/Untitled%204.png)
[31:52](https://youtu.be/QM1iUe6IofM?t=1912)
{: style="color:gray; font-size: 80%; text-align: center;"}

## 객체지향 외의 길

그렇다면, 어떻게 하면 객체지향 프로그래밍이 가져오는 어려움과 복잡성을 피해가는 코드를 작성할 수 있을까?

### 추상화 미루기

위에서 ADT와 같이 데이터와 함수가 명확하게 연관된 경우에는 클래스와 같은 추상화를 만드는 것이 유용할 것이라고 이야기했다. 하지만 성급하게 만들어진 추상화는 높은 확률로 미래에 ADT만큼은 유용하게 쓰이지 않을 것이고, 이는 위에서 이야기한 객체지향 프로그래밍의 단점을 맞이할 가능성을 시사한다.

ADT만큼 데이터와 함수의 연결이 명확한 경우가 아니라면, 데이터와 함수를 따로 두는 전통적인 절차지향 프로그래밍 방식으로 코드를 작성하면서 추상화를 미루는 것이 이런 단점을 피하는 좋은 방법이 될 것이다.

### 긴 함수 / 파일 두려워하지 않기

추상화를 미루게 된다면 자연스레 긴 함수와 긴 파일을 마주하게 될 것이다 - 하지만 이를 무조건적으로 두려워할 필요는 없다. 흔히 함수가 너무 길어지면 이를 분리하는 것이 이상적인 개발자의 습관으로 여겨지지만, 이러한 행위에는 트레이드오프가 존재한다는 점을 명심하자. 

구체적인 예시를 들어보자면, 아래 스크린샷과 같이 `myFunc()` 내의 코드가 너무 길어서 총 4개의 함수로 분리했다고 해 보자. 만약 저 서브 함수들의 정의를 별개로 살펴본다면, 이 함수가 원래 `myFunc()` 내에서 어떤 순서로, 어떤 함수 뒤에 호출되는지에 대한 맥락을 더 이상 찾을 수가 없을 것이다. 이것은 코드의 표면적이 늘어날 때 나타나는 현상의 단적인 예시이고, 객체지향의 원칙을 따라 코드를 클래스 단위로 분해하는 것 또한 비슷한 단점이 발생한다. 이런 단점을 감수하고 긴 함수를 여러 개의 작은 클래스 / 함수로 분리하는 것보다는, 원래의 긴 `myFunc()` 를 그대로 사용하는 것도 충분히 좋은 방법이 될 수 있다.

![[37:38](https://youtu.be/QM1iUe6IofM?t=2258)](/assets/images/2023-05-07/Untitled%205.png)
[37:38](https://youtu.be/QM1iUe6IofM?t=2258)
{: style="color:gray; font-size: 80%; text-align: center; width: 80%; margin: auto;"}

### 가독성 있는 절차지향 코드 작성하기

한 함수에 길게 짜여진 절차지향 코드는 자연스레 유지보수가 어려워지기 십상이지만, 여러 가지 기법으로 절차지향 코드에 대한 성급한 추상화를 피하면서도 유지보수가 용이하도록 관리할 수 있다. 예를 들어서, 함수의 본문에 논리적 블럭 단위로 주석을 달아서 코드 이해를 도울 수 있다. 이러한 방식은 원 함수의 코드 조각들 사이의 맥락을 온전히 보존하면서도, 코드를 읽는 사람의 가독성을 극대화할 수 있을 것이다.

![[38:16](https://youtu.be/QM1iUe6IofM?t=2296)](/assets/images/2023-05-07/Untitled%206.png)
[38:16](https://youtu.be/QM1iUe6IofM?t=2296)
{: style="color:gray; font-size: 80%; text-align: center;"}

만약 코드에 주석만을 달아서 관리하는 것이 어려워진다면, private function을 이용해서 코드를 분리해볼 수 있다. private function 내의 코드는 이 함수 외부에서 사용되지 않는다는 것을 쉽게 판단할 수 있고, 이는 즉 우리가 다뤄야 할 코드의 표면적이 늘어나지 않도록 관리해준다.

![[39:08](https://youtu.be/QM1iUe6IofM?t=2348)](/assets/images/2023-05-07/Untitled%207.png)
[39:08](https://youtu.be/QM1iUe6IofM?t=2348)
{: style="color:gray; font-size: 80%; text-align: center;"}

## 참고한 영상들

- [[Brian Will] Object-Oriented Programming is Bad](https://www.youtube.com/watch?v=QM1iUe6IofM&ab_channel=BrianWill)
- [[Brian Will] Object-Oriented Programming is Good*](https://www.youtube.com/watch?v=0iyB0_qPvWk&t=435s&ab_channel=BrianWill)
- [[Theo - t3.gg] Is OOP EVIL??? Reacting to my favorite dev Youtube video](https://www.youtube.com/watch?v=YpJufWdZFB8&t=1984s&ab_channel=Theo-t3%E2%80%A4gg)
- [[Theo - t3.gg] Big Files Are Good](https://www.youtube.com/watch?v=nYNn_sNcBbM&ab_channel=Theo-t3%E2%80%A4gg)
