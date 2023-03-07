---
layout: single
title: 타입스크립트에서 타입을 알 수 없는 외부 데이터 처리하기
date: 2023-03-06 21:00:00 +0900
categories: software
---

타입스크립트로 개발을 할 때에, 네트워크 요청 등 방식을 통해 타입을 알 수 없는 외부 데이터를 가져오게 되는 경우가 있습니다. 이 글에서는 우선 타입을 알 수 없는 데이터를 다루는 가장 흔한 방법들에 대해서 알아보고, Zod라는 라이브러리를 사용해 앞서 보았던 방식들의 단점을 보완해 데이터를 다룰 수 있는 방법에 대해서 소개해드리고자 합니다.

## 들어가며

타입을 알 수 없는 외부 데이터를 가져오게 되는 경우의 예시를 들어보자면, 아래와 같이 `fetch` 함수를 통해 데이터를 가져오는 경우가 있습니다.

```typescript
async function getUserData(userId: string) {
  return await (await fetch(`/api/user/${userId}`)).json();
}
```

일반적으로 이런 데이터는 정해져있는 스키마가 있지만, API 스펙이 변경되거나 예상치 못한 에러 등 이유로 인해 데이터의 실제 타입은 예상과 다를 수도 있습니다. 그렇다면, 이 함수에서 반환된 데이터를 어떻게 하면 안전하고 편리하게 사용할 수 있을까요?

## 1. `any` 사용

단순히 가장 빠르게 개발하는 것에 초점을 맞추자면... 그냥 타입 정보를 추가하지 않는 것도 방법이 될 수 있습니다. 별도의 타입 정보를 추가하지 않은 상태에서, `getUserData`는 `any` 타입을 반환하게 됩니다.

```javascript
// 아래 함수는 Promise<any>를 반환
async function getUserData(userId: string) {
  return await (await fetch(`/api/user/${userId}`)).json();
}

const data = await getUserData("1");
//    ^? any
console.log(`${data.id}: ${data.name}`);
```

이렇게 **알 수 없는 데이터에 대해 `any` 타입을 사용하는 것은 최대한 피해야 할 방법**입니다. 타입스크립트에서 `any` 타입은 타입을 지정하지 않은 것과 다름이 없습니다. `any` 타입의 데이터를 참조할려고 하면, 타입스크립트가 제공하는 정적 분석, 코드 자동 완성 등의 기능의 도움을 전혀 받을 수 없게 됩니다.

이러한 이유로 인해, ESLint 문서에서는 타입스크립트에 `any`를 사용하는 것에 대해 강력하게 경고하고 있습니다. [관련된 ESLint 문서](https://typescript-eslint.io/rules/no-unsafe-assignment/) 를 살펴보면:

> The `any` type in TypeScript is a dangerous "escape hatch" from the type system. Using `any` disables many type checking rules and is generally best used only as a last resort or when prototyping code.
>
> TypeScript의 `any` 타입은 타입 시스템에서 위험한 “탈출구"입니다. `any`를 사용하면 많은 타입 검사 규칙이 비활성화되므로 일반적으로 최후의 수단으로 또는 코드 프로토타입을 만들 때만 사용하는 것이 가장 좋습니다.

`any`를 실수로라도 사용하는 것을 방지하기 위해서는, ESLint에서 [@typescript-eslint/recommended-requiring-type-checking](https://typescript-eslint.io/linting/configs/#recommended-requiring-type-checking) 설정을 사용할 수 있습니다. 이 설정은 [no-unsafe-assignment](https://typescript-eslint.io/rules/no-unsafe-assignment/) 규칙 등 any에 대한 참조에 대해 에러를 발생시키는 다양한 규칙들이 포함되어 있습니다.

![recommended-requiring-type-checking 설정 포함 후 any를 사용할 떄 에러가 나는 모습](/assets/images/2023-03-06/1-linter.png)
_recommended-requiring-type-checking 설정 포함 후 `any` 타입을 사용할 떄 에러가 나는 모습_
{: style="color:gray; font-size: 80%; text-align: center;"}

## 2. Type Narrowing

타입스크립트의 Type Narrowing 기능은 런타임에 타입을 알 수 없는 데이터의 타입을 확인할 수 있는 방법 중 하나입니다. 가장 흔히 쓰이는 방법은 `typeof` 연산자와 `if`문을 같이 활용하는 것입니다. 이를 활용하면 외부에서 들어온 `data`의 타입이 기대와 동일한지를 런타임에 올바르게 검증하고, 예기치 못한 타입이 내려왔을 때에도 올바르게 대응할 수 있습니다.

아래는 `typeof` 연산자와 `if`문을 활용한 Type Narrowing의 예시입니다.

```typescript
let data: any;

if (typeof data === "string") {
  console.log(data.length);
  //          ^?let data: string
} else if (typeof data === "number") {
  console.log(data.toFixed());
  //          ^?let data: number
}
```

이렇게 Type Narrowing을 통해 데이터의 타입을 검증할 수 있지만, 이 방식에도 큰 단점이 있습니다.

먼저, Type Narrowing은 객체의 속성에 대해서 잘 동작하지 않습니다. 예를 들어서, 아래 코드처럼 `data.id`의 타입이 string이라는 것을 검증했다고 하더라도, `data` 와 `data.id` 모두 여전히 `any` 타입으로 추론됩니다. (_이는 타입스크립트의 설계적 한계로 발생하는 이슈입니다. [관련된 StackOverflow 질문](https://stackoverflow.com/a/57929239)도 확인해 보세요!)_

```typescript
let data: any;
if (typeof data.id === "string") {
  console.log(data.id);
  //               ^?any
}
```

뿐만 아니라, **`typeof`를 이용해 Type Narrowing을 하는 코드를 작성하는 것은 필드가 많아질수록 유지보수하기 매우 어려워집니다**. 큰 규모의 프로젝트에서는 수십 개의 필드를 가지고 있는 객체를 다루게 되는데, 이런 경우에도 모든 필드에 대해 `typeof`를 이용한 Type Narrowing을 적용하기는 현실적으로 불가능합니다.

## 3. Type Assertion

Type Assertion은 조금 더 타입스크립트의 타입 시스템을 직접적으로 활용하는 방식입니다. 이는 아래와 같이 타입스크립트의 `type`이나 `interface` 키워드로 외부 데이터의 타입을 정의하고, `as` 키워드를 통해 데이터의 타입을 직접 명시하는 방식입니다.

```typescript
type UserAPIResponse = {
  id: string;
  name: string;
};
async function getUserData(userId: string) {
  return (await (
    await fetch(`/api/user/${userId})`)
  ).json()) as UserAPIResponse;
}

const data = await getUserData("1");
//    ^?const data: UserAPIResponse
// 이렇게 하면 id, name에 대한 자동완성이 가능!
console.log(`${data.id}: ${data.name}`);
```

이 방식을 사용하면 `data`값의 타입이 `UserAPIResponse`로 지정되어서, 자동완성이나 정적 분석 등 타입스크립트가 제공하는 기능들을 개발 중에 충분히 활용할 수 있습니다.

하지만 이 방식에도 치명적인 단점이 있는데요, 바로 **데이터가 실제로 지정된 타입과 일치하는지 런타임에 전혀 검증이 되지 않는다**는 것입니다. 외부 데이터의 타입이 기대했던 것과 다르다고 해도, `as`키워드로 Type Assertion을 하는 시점에 에러가 발생하지 않고, 런타임에서 필드를 참조할 때에 엉뚱한 곳에서 에러가 발생하게 됩니다.

이는 타입스크립트로 작성한 타입 정보가 코드가 컴파일되어 자바스크립트로 변환되기 이전에만 존재하기 때문에 그렇습니다. 타입스크립트 코드를 작성하더라도, 실제로 런타임에서 실행되는 것은 순수한 자바스크립트 코드 뿐인데요, 따라서 `as` 키워드 또한 런타임에는 삭제되어 실행됩니다. [Typescript 공식 문서의 "Erased Types" 항목](https://www.typescriptlang.org/docs/handbook/2/basic-types.html#erased-types)에서도, 이와 관련된 정보를 확인할 수 있습니다.

> Remember: Type annotations never change the runtime behavior of your program.
>
> 기억하세요: 타입 어노테이션은 프로그램의 런타임 동작을 변경하지 않습니다.

![Type Assertion이 컴파일 후의 자바스크립트에는 전혀 남아있지 않은 모습을 확인할 수 있음](/assets/images/2023-03-06/2-compiler.png)
_Type Assertion이 컴파일 후의 자바스크립트에는 전혀 남아있지 않은 모습을 확인할 수 있음. [TS Playground](https://www.typescriptlang.org/play?ts=4.9.5#code/C4TwDgpgBAqgzhATgQQAoEkBKE5gPYB2CUAvFAN4BQUUAlgCYBcUcwitBA5gNzVQEBDALYRmrdl14BfXgLggCAYygAzAK5LgtQlE4Rg8JABEBwAQAo1CROiYs2HTgEoKfRPrWICUcwIDuArTAPnw0-oHBKvqKABbmAAYA9AJgtIlWSIkAJOQZNvRSTvFOfE4AdABWcITmTi5ysNZoWDj4RBDSlJRyCsrqmtreQoEEta40ioSsUMFk4UG6+oaIJmbmAOQAjOtOvBNTeAA2EGWHeJzmwGUMu5RS1EA)_
{: style="color:gray; font-size: 80%; text-align: center;"}

즉, Type Assertion은 개발 중에는 타입스크립트의 타입 시스템을 잘 활용하도록 돕지만, 그렇게 작성된 코드가 런타임에도 에러 없이 안전하게 실행될 것이라는 건 보장할 수 없습니다.

## 4. Zod

마지막으로 소개드릴 방법은 [Zod](https://github.com/colinhacks/zod) 라이브러리를 사용하는 것입니다. Zod는 타입스크립트를 우선하는 스키마 선언 및 검증 라이브러리입니다. 이 라이브러리를 사용하면 앞서 보았던 여러 방식의 문제점들을 쉽게 해결할 수 있는데요, 먼저 아래 예시를 보겠습니다.

```typescript
import { z } from "zod";

// Zod에서 제공하는 메소드를 이용해 스키마를 생성
const UserAPISchema = z.object({
  id: z.string(),
  name: z.string(),
});
async function getUserData(userId: string) {
  const fetched = await (await fetch(`/api/user/${userId}`)).json();
  // .parse 메소드로 데이터를 검증하고, 검증되어 타입 정보가 붙은 데이터를 반환
  return UserAPISchema.parse(fetched);
}
const data = await getUserData("1");
//    ^?{ id: string; name: string; }
console.log(`${data.id}: ${data.name}`);
```

Zod를 사용할 때에는, 먼저 위 예시의 `UserAPISchema`처럼 Zod에서 제공하는 메소드를 이용해 스키마를 생성합니다. 그 후 생성된 스키마를 통해 `.parse`를 호출하면, 데이터가 스키마와 일치하는지 검증하고, 검증이 성공하면 타입 정보가 있는 데이터를 반환합니다. Zod 스키마를 사용해 데이터를 검증하면, 앞서 보았던 다른 방법들의 문제점을 쉽게 해결할 수 있습니다.

- if문을 통해 Type narrowing을 할 때와는 달리, Zod 스키마는 선언적으로 작성되어 코드를 읽는 사람이 스키마의 구조를 쉽게 파악할 수 있습니다. 또한, 스키마를 작성하는 것도 타입스크립트 타입을 작성하는 것과 유사하기 때문에, 타입스크립트를 사용하는 개발자라면 쉽게 익힐 수 있습니다.
- `.parse`에서 반환된 데이터는 스키마와 일치하는 타입스크립트 타입을 가지고 있기 때문에, 타입 정보를 개발 중에 활용할 수 있습니다. 위 코드에서도 `data`의 타입이 자동으로 `{ id: string; name: string; }`으로 추론됩니다.
- `.parse`에서 반환된 데이터는 `as` 키워드를 사용한 Type Assertion과는 다르게, 데이터가 스키마와 일치하는지 실제로 런타임에서 검증을 거치기 때문에, 런타임에도 안심하고 데이터를 다룰 수 있습니다.

또한, 만약 Zod 스키마 대신 타입스크립트 타입을 직접 사용해야 한다면, 이 또한 Zod에서 제공하는 [Type inference](https://github.com/colinhacks/zod#type-inference) 기능을 활용해 Zod 스키마에서 타입스크립트 타입을 쉽게 추출할 수 있습니다. 즉, Zod 스키마 하나만 작성하면 타입스크립트 타입을 중복으로 작성할 필요 또한 없습니다.

```typescript
const A = z.string();
type A = z.infer<typeof A>; // string
```

### 번외. Zod vs Joi

Zod와 Joi는 비슷하게 데이터 검증을 하는 라이브러리고, 실제로 Zod가 있기 전에는 Joi를 많이 사용했지만, 지금은 Zod가 상대적으로 더 많이 사용되고 있습니다.

![2023년 3월 기준으로는, Zod의 Github 스타 수가 Joi를 거의 따라잡았습니다](/assets/images/2023-03-06/4-zod-star.png)
_2023년 3월 기준으로는, Zod의 Github 스타 수가 Joi를 거의 따라잡았습니다. [참고: star-history](https://star-history.com/#colinhacks/zod&hapijs/joi&Date)_
{: style="color:gray; font-size: 80%; text-align: center;"}

Zod가 Joi 대비해서 인기를 얻은 큰 이유 중 하나는 Zod의 훌륭한 타입스크립트 지원입니다. Joi는 데이터에 대한 검증을 해 줄 때 그 검증으로 얻은 정보가 타입스크립트 타입으로 남지 않습니다. 반면에 Zod에서 데이터를 파싱한 후에는 데이터의 정확한 타입을 타입스크립트 타입으로도 볼 수 있습니다. 때문에, 타입스크립트 생태계 내에서 Zod를 사용하면 (Joi를 사용하는 것과 비교해서) 개발자 생산성을 크게 개선시킬 수 있다고 생각합니다.

Zod와 다른 데이터 검증 라이브러리들 간의 비교는 [Zod 레포지토리의 Comparison 항목](https://github.com/colinhacks/zod#comparison) 에서도 확인해볼 수 있습니다.

## 마치며

위와 같이 외부에서 들어온 데이터를 검증하는 여러가지 방식에 대해서 살펴보았습니다. 저는 그동안의 타입스크립트 프로젝트에서는 앞 세 가지 방식을 혼용해서 외부 데이터를 다뤄 왔었는데, Zod를 사용하고 나서는 외부 데이터를 처리하는 것이 이전에 비해 아주 안전하고 편리해진 것을 느끼고 있습니다.

혹시 여러분의 타입스크립트 프로젝트도 외부에서 들어온 데이터의 타입을 어떻게 하면 올바르게 검증하고 사용할 수 있을까 고민이 된다면, Zod를 한번 사용해보시는 것을 추천드리고 싶습니다!

## 참고자료

- [Types are just data - What about `any`?](https://type-level-typescript.com/types-are-just-data#what-about-any)
- [You Might Be Using Typescript Wrong](https://youtu.be/RmGHnYUqQ4k)
- [검증하지 말고 파싱하라](https://eatchangmyeong.github.io/2022/12/04/parse-don-t-validate.html)
