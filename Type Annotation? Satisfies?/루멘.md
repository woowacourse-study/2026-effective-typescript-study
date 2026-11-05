8월 25일 연어 프로젝트를 진행하다 발생한 일이였다...

나는 연어 서비스의 노선 서비스 선택 화면을 구현하고 있었다. 해당 화면에 필요한 API 스키마는 완성이 되었지만, 실제 API는 아직 개발되지 않아 Mock Data를 이용해서 구현해야 했다.

다음 코드는 버스 노선들을 보여주는 `RouteList` 컴포넌트다.

```tsx
export interface Route {
  id: string;
  displayName: string;
  startStopName: string;
  endStopName: string;
  status: "FORECAST_READY" | "PREPARING";
}

interface RouteListProps {
  routes: Route[];
}

export function RouteList({ routes }: RouteListProps) {
  return (
    <ul>
      {routes.map((route) => (
        <RouteListItem
          key={route.id}
          displayName={route.displayName}
          startStopName={route.startStopName}
          endStopName={route.endStopName}
        />
      ))}
    </ul>
  );
}
```

서버에서 불러온 노선 배열을 받아, 배열의 길이만큼 `RouteListItem`을 렌더링하는 컴포넌트다.

아직 API가 없었기 때문에 페이지에서 Mock Data를 만들어 전달했다.

```tsx
export function RouteSelectPage() {
  const routesMock = [
    {
      id: "204000057",
      displayName: "3330",
      startStopName: "도촌동9단지앞",
      endStopName: "안양역",
      status: "PREPARING",
    },
    {
      id: "234000050",
      displayName: "1650",
      startStopName: "구리수택차고지",
      endStopName: "안양역",
      status: "FORECAST_READY",
    },
  ];

  return (
    <main>
      <RouteList routes={routesMock} />
    </main>
  );
}
```

하지만 타입체크를 통과하지 못했고, 다음과 같은 에러 메시지를 만나게 되었다.

> Type 'string' is not assignable to type '"FORECAST_READY" | "PREPARING"'.

분명 `status`에는 `"PREPARING"`과 `"FORECAST_READY"`를 넣었는데, 왜 `string`이라는 것일까?

에디터에서 `routesMock`의 타입을 확인해보니 다음과 같이 추론되고 있었다.

```ts
// routesMock의 추론된 타입
{
  id: string;
  displayName: string;
  startStopName: string;
  endStopName: string;
  status: string;
}
[];
```

문제는 실제로 넣은 값이 아니라, 그 값으로부터 추론된 타입에 있었다.

`Route`의 `status`는 두 문자열만 허용하지만, `routesMock`의 `status`는 모든 문자열을 허용하는 `string`으로 추론된 것이다. `string`에는 `"RUNNING"`이나 `"UNKNOWN"` 같은 값도 들어갈 수 있으므로, TypeScript는 이 배열을 `Route[]`로 사용할 수 없다고 판단했다.

## 그렇다면 어떻게 해결해야 할까?

이번에는 다음 세 가지 방법을 비교해보려고 한다.

1. `Route[]`로 타입 어노테이션 해주기
2. `as Route[]`로 타입 단언하기
3. `satisfies Route[]`로 타입 검사하기

이후 예제에서는 비교를 위해 노선 하나만 사용했다.

### Route[]로 타입 어노테이션 해주기

실제로 내가 처음 시도했던 방법은 타입 어노테이션이었다.

```ts
const routesMock: Route[] = [
  {
    id: "204000057",
    displayName: "3330",
    startStopName: "도촌동9단지앞",
    endStopName: "안양역",
    status: "PREPARING",
  },
];
```

이렇게 하면 오류가 해결된다.

`Route[]`라는 타입을 명시했기 때문에, TypeScript는 배열의 각 요소가 `Route`에 맞는지 검사한다. `status` 역시 단순한 `string`으로 추론하지 않고, `"FORECAST_READY" | "PREPARING"`이라는 타입에 맞는 값인지 확인한다.

필수 속성을 빠뜨리거나 잘못된 상태 값을 넣으면 선언하는 시점에 오류가 발생한다.

```ts
const routesMock: Route[] = [
  {
    id: "204000057",
    displayName: "3330",
    startStopName: "도촌동9단지앞",
    endStopName: "안양역",
    status: "RUNNING", // 타입 오류
  },
];
```

다만 이 방식에서 변수의 타입은 우리가 명시한 `Route[]`가 된다.

현재 데이터에는 `"PREPARING"`만 넣었지만, 배열 요소의 `status` 타입은 `"FORECAST_READY" | "PREPARING"`이다. 이후 다른 상태의 노선을 추가하거나 기존 노선의 상태를 변경하는 것도 허용된다.

이대로 사용해도 괜찮을까?

이번처럼 `RouteList`에 전달할 데이터를 만드는 목적이라면 충분히 괜찮다. 이 배열을 `Route`들을 담는 배열로 사용하겠다는 의도가 명확하기 때문이다.

### as Route[]로 타입 단언하기

두 번째 방법은 `as Route[]`로 타입을 단언하는 것이다.

```ts
const routesMock = [
  {
    id: "204000057",
    displayName: "3330",
    startStopName: "도촌동9단지앞",
    endStopName: "안양역",
    status: "PREPARING",
  },
] as Route[];
```

이렇게 해도 타입 오류는 해결된다. TypeScript가 해당 표현식을 `Route[]`로 취급하기 때문이다.

결과만 보면 타입 어노테이션과 비슷해 보인다. 하지만 데이터를 검사하는 방식에는 차이가 있다.

타입 어노테이션이 “이 값이 `Route[]`에 맞는지 확인해줘”라는 의미라면, 타입 단언은 “이 값은 `Route[]`라고 내가 보장할게”에 가깝다.

다음 코드를 보면 차이가 드러난다.

```ts
const routesMock = [
  {
    id: "204000057",
  },
] as Route[];
```

`Route`에 필요한 속성이 대부분 빠져 있지만, 이 코드는 타입체크를 통과한다.

문제는 이후에 이 데이터를 사용할 때 발생한다.

```ts
routesMock.forEach((route) => {
  route.displayName.toUpperCase();
});
```

컴파일러는 `route`를 `Route`로 취급하므로, `displayName`이 문자열이라고 판단한다. 하지만 실제 객체에는 `displayName`이 없다. 따라서 실행하면 `undefined`에서 `toUpperCase()`를 호출하려고 하면서 오류가 발생한다.

타입 단언은 실제 데이터를 바꾸거나 빠진 속성을 채워주지 않는다.

물론 `as`를 붙인다고 모든 단언이 허용되는 것은 아니다. TypeScript는 서로 충분히 관련되지 않은 타입 사이의 단언은 거부한다. 하지만 위 예제처럼 필수 속성이 빠져 있어도 허용하는 경우가 있다.

직접 작성하는 Mock Data라면 누락이나 오타를 발견하는 것이 중요하다. 따라서 이 사례에서는 타입 단언보다 타입 어노테이션을 사용하는 편이 적절하다고 생각한다.

### satisfies로 타입 검사하기

세 번째 방법은 `satisfies`를 사용하는 것이다.

```ts
const routesMock = [
  {
    id: "204000057",
    displayName: "3330",
    startStopName: "도촌동9단지앞",
    endStopName: "안양역",
    status: "PREPARING",
  },
] satisfies Route[];
```

`satisfies`는 표현식이 지정한 타입의 조건을 만족하는지 검사한다. 따라서 필수 속성이 빠져 있거나 `status`에 잘못된 값을 넣으면 오류가 발생한다.

그렇다면 타입 어노테이션과는 무엇이 다를까?

타입 어노테이션은 변수의 타입을 `Route[]`로 지정한다. 반면 `satisfies`는 `Route[]`에 맞는지 검사하면서, 작성한 데이터로부터 얻은 구체적인 추론 결과를 사용할 수 있게 한다.

위 예제에서 배열 요소는 다음과 같이 추론된다.

```ts
{
  id: string;
  displayName: string;
  startStopName: string;
  endStopName: string;
  status: "PREPARING";
}
```

타입 어노테이션을 사용했을 때 `status`가 두 상태의 유니온이었다면, 이 예제에서는 실제로 작성한 `"PREPARING"`으로 추론된다.

다만 `satisfies`가 모든 속성을 리터럴 타입으로 유지하는 것은 아니다. 위에서도 `id`와 `displayName`은 여전히 `string`이다. `status`는 `Route[]`가 제공하는 리터럴 유니온의 문맥에서 추론되기 때문에 문자열 리터럴 타입이 유지된다.

처음 작성한 Mock Data처럼 두 상태를 모두 넣으면, 배열 요소의 `status`에는 두 상태가 모두 반영된다. 따라서 원래 예제에서는 타입 어노테이션과 비교했을 때 사용상의 차이가 크지 않다.

결국 타입 어노테이션이 잘못된 방법이고 `satisfies`가 무조건 더 좋은 방법인 것은 아니다. 변수를 `Route[]`로 다루고 싶다면 타입 어노테이션을, 타입 조건을 검사하면서 데이터의 구체적인 추론 결과도 활용하고 싶다면 `satisfies`를 사용할 수 있다.

## 객체 속성은 왜 기본형 타입으로 추론이 될까?

오류를 해결하는 방법은 알게 되었다. 그런데 처음에 `routesMock`의 `status`가 `string`으로 추론된 이유는 무엇일까?

TypeScript는 객체 속성이 변경될 수 있다고 보기 때문이다. 별도의 타입 문맥이 없다면, 문자열이나 숫자 리터럴로 초기화한 객체 속성을 대응하는 기본형 타입으로 넓혀서 추론하는 경우가 많다.

다음 코드를 보자.

```ts
const obj = { x: 1 };

// obj: { x: number }
```

`x`의 초기값은 `1`이지만, 타입은 리터럴 타입 `1`이 아니라 `number`로 추론된다. 덕분에 이후 다른 숫자로 변경할 수 있다.

```ts
obj.x = 2; // 가능
```

만약 `x`의 타입을 `1`로 추론했다면, `2`를 대입하는 코드도 타입 오류가 되었을 것이다. 객체 속성은 나중에 변경하면서 사용하는 경우가 많기 때문에, 매번 이런 오류가 발생한다면 불편할 것이다.

여기서 `const`로 선언했으니 값이 바뀌면 안 되는 것 아닌가 하는 생각이 들 수도 있다.

하지만 `const`는 변수에 다른 값을 재할당하지 못하게 할 뿐, 객체의 속성까지 변경하지 못하게 하지는 않는다.

```ts
const obj = { x: 1 };

obj = { x: 2 }; // 오류: 변수 재할당
obj.x = 2; // 가능: 객체 속성 변경
```

`routesMock`도 마찬가지다. `const`로 선언했더라도 배열 안의 객체 속성은 변경할 수 있다. 따라서 별도의 문맥이 없다면 `status`는 `"PREPARING"`이나 `"FORECAST_READY"`에 고정되지 않고 `string`으로 넓혀서 추론된다.

## 리터럴 타입을 그대로 유지하고 싶다면?

객체 속성을 넓은 기본형 타입이 아닌, 특정 리터럴 타입으로 유지하고 싶을 수도 있다. 그런 경우에는 `as const`를 사용할 수 있다.

```ts
const obj = {
  x: 1 as const,
};

// obj: { x: 1 }
```

이제 `x`는 `number`가 아니라 `1`이라는 타입을 가진다.

```ts
obj.x = 1; // 가능
obj.x = 2; // 타입 오류
```

여기서는 `x`의 값에만 const 단언을 적용했으므로, `x` 속성 자체가 `readonly`가 된 것은 아니다. 대입할 수 있는 값의 타입이 `1`로 제한된 것이다.

객체 전체에 const 단언을 적용하면 속성도 읽기 전용으로 추론된다.

```ts
const obj = {
  x: 1,
} as const;

// obj: { readonly x: 1 }

obj.x = 1; // 타입 오류: 읽기 전용 속성
```

앞에서 사용한 `as Route[]`와 문법은 비슷하지만 목적은 다르다. `as Route[]`는 개발자가 지정한 타입으로 취급하도록 하는 단언이고, `as const`는 작성한 값의 리터럴 타입을 유지하면서 객체 속성과 배열을 읽기 전용으로 추론하도록 하는 단언이다.

따라서 Mock Data 배열 전체에 `as const`를 붙이면, `status`의 타입 넓히기는 막을 수 있지만 배열도 읽기 전용 튜플이 된다. 현재 `RouteList`는 변경 가능한 `Route[]`를 받으므로, 그대로 전달하면 다른 타입 오류가 발생한다.

이번 오류는 Mock Data에 잘못된 값을 넣어서 발생한 것이 아니었다. 내가 생각한 타입과 TypeScript가 추론한 타입이 달랐기 때문에 발생한 것이었다.

처음에는 오류를 없애는 데 집중했지만, 같은 오류를 해결하더라도 타입을 명시하는 것과 단언하는 것, 조건을 만족하는지 검사하는 것은 서로 다른 선택이라는 것을 알게 되었다. 앞으로는 타입 오류가 발생했을 때 실제 값뿐 아니라, 그 값이 어떤 타입으로 추론되고 있는지도 함께 확인해보려고 한다.
