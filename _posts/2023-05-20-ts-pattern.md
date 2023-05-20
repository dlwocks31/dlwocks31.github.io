---
layout: single
title: 타입스크립트에서 ts-pattern과 패턴 매칭 사용해보기
date: 2023-05-20 16:00:00 +0900
categories: software
---


이 글에서는 타입스크립트 환경에서 사용할 수 있는 ts-pattern이라는 패턴 매칭 라이브러리에 대해 간단히 소개하려 합니다. 

패턴 매칭은 흔히 함수형 프로그래밍 언어에서 쓰이는 기법으로, 주어진 데이터의 구조에 따라 분기하는 코드를 선언적으로 작성할 수 있게 돕습니다. ts-pattern과 패턴 매칭을 사용해, 동일한 기능의 절차지향 코드보다 더욱 간결하고 유지보수가 용이한 코드를 만들어낼 수 있습니다. 

## 상태에 따른 분기 예시

타입스크립트에서 ts-pattern을 활용한 패턴 매칭의 장점을 생각해보기 위해, 먼저 아래와 같이 여러 개의 상태를 가질 수 있는 `fetchState` 라는 데이터를 생각해 봅시다. 이 값의 상태에 따라 분기처리를 하는 가장 일반적인 방법은 `switch` 또는 `if` 문을 사용하는 것입니다.

```tsx
declare let fetchState:
  | { status: { label: "loading" }}
  | { status: { label: "success" }, data: string }
  | { status: { label: "error" }, message: string };

// switch 사용
switch (fetchState.status.label) {
  case "loading":
    console.log("loading..");
    break;
  case "success":
    console.log("success! data is ", fetchState.data);
    break;
  case "error":
    console.error("error: ", fetchState.message);
    break;
}

// if 사용
if (fetchState.status.label === "loading") {
  console.log("loading..");
} else if (fetchState.status.label === "success") {
  console.log("success! data is ", fetchState.data);
} else if (fetchState.status.label === "error") {
  console.error("error: ", fetchState.message);
}
```
이 코드는 분명 `fetchState` 값에 따라 분기하게 되는 올바른 코드이지만, 편의성과 유지보수성 측면에서 아쉬운 점들을 찾아볼 수 있습니다.
- 타입 변경에 취약합니다. 만약 fetchState의 타입에 또 다른 종류의 상태(ex. `status: { label: "idle" }` ) 가 추가되어도, 위 switch문과 if문에서는 컴파일 에러가 발생하지 않기 때문에, 개발자가 상태를 추가한 후 위와 같이 상태에 따라 분기하는 코드를 일일히 다시 살펴야 합니다.
- 특정 상태를 선택한 후의 타입 추론이 되지 않을 수 있습니다. 아래와 같이 분명 개발자는 `fetchState.data` 필드가 존재한다는 사실을 알 수 있음에도 불구하고, 타입스크립트의 타입 시스템에서는 이러한 사실이 추론되지 않습니다. ([관련 Typescript 이슈](https://github.com/microsoft/TypeScript/issues/18758))
![Untitled](/assets/images/2023-05-20/failed_type.png)



## ts-pattern으로 다시 쓰기

위에서 switch 와 if 문을 사용해 작성한 코드를 ts-pattern을 사용한 패턴 매칭 코드로 다시 작성해보았습니다.

```tsx
match(fetchState)
  .with({ status: { label: "loading" } }, () => console.log("loading.."))
  .with({ status: { label: "success" } }, ({ data }) =>
    console.log("success! data is ", data)
  )
  .with({ status: { label: "error" } }, ({ message }) =>
    console.error("error: ", message)
  )
  .exhaustive();
```

ts-pattern을 사용한 코드의 대략적인 구조 자체는 switch문을 사용한 코드와 유사합니다. 분기의 시작에는 ts-pattern의 `match` 객체를 사용하고, 각 케이스마다 `with` 함수를 이용해 표시하며, 맨 마지막에 `exhaustive()` 함수를 호출합니다.

이 코드는 원래 코드에서 보았던 문제점들을 해결합니다.

- 타입 변경 시에도 타입스크립트의 도움을 최대한 받을 수 있습니다. ts-pattern의 `.exhaustive()` 는 `fetchState` 의 가능한 값을 처리하는 `with` 함수가 없다면 컴파일 에러를 발생시킵니다. 따라서 아래와 같이 fetchState에 가능한 상태를 하나 추가하면, 컴파일 에러가 발생해 개발자가 쉽게 수정해야 할 부분을 포착할 수 있게 됩니다.

![Untitled](/assets/images/2023-05-20/exh.png)

- 타입 추론이 쉬워집니다. `with` 함수의 콜백에는 패턴에 맞는 데이터가 타입 추론이 된 채로 넘어오기 때문에, 아래와 같이 원래 코드에서는 올바르게 타입 추론이 되지 않던 `data` 와 `message` 모두 올바른 타입 정보와 함께 사용할 수 있습니다.

![Untitled](/assets/images/2023-05-20/type.png)

만약 디폴트 분기를 추가하고 싶다면, `exhaustive()` 대신 `otherwise()` 를 사용해 명시되지 않은 케이스에 대한 처리를 한번에 할 수 있습니다.

```ts
match(fetchState)
  .with({ status: { label: "loading" } }, () => console.log("loading.."))
  .with({ status: { label: "success" } }, ({ data }) =>
    console.log("success! data is ", data)
  )
  .with({ status: { label: "error" } }, ({ message }) =>
    console.error("error: ", message)
  )
  .otherwise(() => console.error("other status"));
```

## 응용하기: State Reducer

위에서 패턴 매칭과 ts-pattern을 사용하는 간단한 예시를 보았습니다. 패턴 매칭과 ts-pattern의 진가는 로직이 복잡해질 때 더욱 빛을 발하는데요, 그 예시로 현재 상태와 이벤트를 조합해서 새로운 상태를 만드는 state reducer의 구현을 살펴보겠습니다.

아래와 같이 총 네 가지의 상태와 네 가지의 이벤트가 있다고 합니다.

```ts
type State =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: string }
  | { status: 'error'; error: Error };

type Event =
  | { type: 'fetch' } // 상태가 `loading` 이 아닌 모든 상태 → `loading` 으로 변화
  | { type: 'success'; data: string } // 상태가 `loading` → `success` 로 변화
  | { type: 'error'; error: Error } // 상태가 `loading` → `error` 로 변화
  | { type: 'cancel' }; // 상태가 `loading` → `idle` 로 변화
```

이 네 가지 이벤트와 네 가지 상태의 조합을 그림으로 그려보면 아래와 같습니다.

![Untitled](/assets/images/2023-05-20/diagram.png)

이처럼 다소 복잡할 수 있는 이벤트와 상태의 조합 또한, 아래와 같이 ts-pattern을 사용해서 가능한 이벤트와 상태의 조합, 그리고 각 조합의 다음 상태 모두 선언적으로 표현할 수 있습니다.

```ts
import { match, P } from 'ts-pattern';

const reducer = (state: State, event: Event): State =>
  match<[State, Event], State>([state, event])
    .with(
      [{ status: 'loading' }, { type: 'success' }],
      ([, event]) => ({ status: 'success', data: event.data })
    )
    .with(
      [{ status: 'loading' }, { type: 'error', error: P.select() }],
      (error) => ({status: 'error', error })
    )
    .with(
      [{ status: P.not('loading') }, { type: 'fetch' }],
      () => ({ status: 'loading' })
    )
    .with(
      [{ status: 'loading' }, { type: 'cancel' }],
      () => ({ status: 'idle' })
    )
    .with(P._, () => state)
    .exhaustive();
```

## 레퍼런스

- [Github: gvergnaud/ts-pattern](https://github.com/gvergnaud/ts-pattern)