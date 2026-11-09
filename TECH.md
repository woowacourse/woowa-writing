# 다형성 컴포넌트를 직접 만들며 마주친 TypeScript 타입 문제들

React에서 공통 UI 컴포넌트를 만들다 보면 같은 디자인으로 서로 다른 HTML 요소를 렌더링해야 할 때가 있다. 렌더링할 요소를 `as`로 선택할 수 있게 만들었더니, 이번에는 선택한 요소에 맞춰 Props 타입도 바꿔야 했다.

이 글은 React와 TypeScript로 공통 UI 컴포넌트를 만들어 본 개발자를 대상으로 한다. 공통 스타일을 처리하는 `View`와 이를 확장한 `Button`을 예로 들어, 요소별 Props를 연결하고 이름이 겹치는 Props의 우선순위를 정하는 과정을 살펴본다.

## 1. 공통 컴포넌트에서 HTML 요소가 고정되는 문제

### 1.1 같은 디자인, 다른 HTML 요소

같은 버튼 디자인으로 폼을 제출하고 책 목록으로 이동한다고 가정해 보자. 아래 공통 `Button`은 두 경우 모두 `<button>`으로 렌더링된다.

```tsx
<Button onClick={handleSubmit}>
  제출하기
</Button>

<Button onClick={() => navigate('/books')}>
  책 목록 보기
</Button>
```

첫 번째 컴포넌트는 사용자의 동작을 실행하므로 `<button>` 요소가 적절하다.

반면 두 번째 컴포넌트는 다른 페이지로 이동하는 기능을 수행한다. 이러한 경우에는 `<a href="/books">` 요소가 더 적절하다.

HTML 요소는 단순히 화면에 표시되는 모양만 결정하지 않는다.

`<a>` 요소는 링크 주소를 복사하거나 새 탭에서 여는 등 브라우저가 기본적으로 제공하는 탐색 기능을 사용할 수 있다. 반면 `<button>` 요소는 동작을 실행하는 데 사용되며, 폼 내부에서는 제출 버튼으로 동작할 수도 있다.

따라서 같은 디자인을 사용하더라도 목적에 맞는 HTML 요소를 선택하는 것이 중요하다.

이러한 요구사항을 해결하려면 공통 컴포넌트가 특정 HTML 요소에 종속되지 않도록 만들어야 한다.

### 1.2 HTML 속성을 직접 정의해야 하는 문제

HTML 요소가 고정되어 있다는 것 외에도 문제가 하나 더 있었다.

공통 컴포넌트에서 HTML의 기본 속성을 사용하려면 Props 타입을 정의해야 한다.

```tsx
type ButtonProps = {
  children: React.ReactNode;
  onClick?: () => void;
  type?: 'button' | 'submit' | 'reset';
  disabled?: boolean;
  name?: string;
  value?: string;
};
```

처음에는 필요한 속성을 하나씩 추가할 수 있다. 하지만 컴포넌트를 사용하는 곳이 늘어나면 지원해야 할 속성도 많아진다.

이미 React의 타입 정의에 존재하는 HTML 속성을 다시 작성하는 것은 중복 작업이다. 또한 속성을 누락하거나 실제 HTML 요소와 다른 타입으로 정의할 가능성도 있다.

결국 해결하고 싶은 문제는 두 가지였다.

1. 사용하는 곳에서 렌더링할 HTML 요소를 결정할 수 있어야 한다.
2. 선택한 HTML 요소에 따라 적절한 Props를 사용할 수 있어야 한다.

단순히 렌더링하는 요소만 변경하는 것으로는 충분하지 않았다.

---

## 2. HTML 요소의 Props를 재사용하는 방법

### 2.1 ComponentPropsWithoutRef

React는 HTML 요소나 React 컴포넌트가 사용하는 Props 타입을 가져올 수 있도록 `ComponentPropsWithoutRef`를 제공한다.

```tsx
type ButtonProps =
  React.ComponentPropsWithoutRef<'button'>;
```

이렇게 정의하면 `<button>` 요소에서 사용할 수 있는 속성을 직접 작성하지 않아도 된다.

```tsx
<Button type="submit" disabled>
  제출하기
</Button>
```

반대로 `<button>` 요소에 존재하지 않는 `href`를 전달하면 TypeScript가 오류를 표시한다.

```tsx
<Button href="/books">
  책 목록 보기
</Button>
```

`href`는 `<a>` 요소에서 사용하는 속성이기 때문이다.

`ButtonHTMLAttributes<HTMLButtonElement>`로도 버튼의 HTML 속성을 표현할 수 있다. 버튼의 `type`을 `'button' | 'submit' | 'reset'`으로 제한하는 것은 두 타입 모두 같다.

이번 구현에서는 `as`에 HTML 요소 이름과 React 컴포넌트를 모두 전달할 수 있도록 설계했다. `ComponentPropsWithoutRef<T>`를 사용하면 두 경우 모두 같은 방식으로 대상의 Props를 가져올 수 있다.

### 2.2 as 패턴과 asChild 패턴

렌더링할 HTML 요소를 변경하는 방법에는 대표적으로 `as` 패턴과 `asChild` 패턴이 있다.

`as` 패턴은 컴포넌트에 전달한 `as` 속성을 기준으로 렌더링할 요소를 결정한다.

```tsx
<Button as="button" type="submit">
  제출하기
</Button>

<Button as="a" href="/books">
  책 목록 보기
</Button>
```

반면 `asChild` 패턴은 자식 요소를 활용한다.

```tsx
<Button asChild>
  <a href="/books">
    책 목록 보기
  </a>
</Button>
```

`asChild` 방식에서는 Button이 별도의 `<button>` 요소를 만드는 대신 자식 요소에 스타일이나 이벤트 핸들러 등의 Props를 병합한다.

이 방식은 자식 요소를 직접 작성할 수 있다는 장점이 있지만, Props와 이벤트 핸들러를 병합하고 ref를 전달하는 과정에서 추가적인 구현이 필요하다.

이번에는 `as` 패턴을 선택했다.

`as`에 전달한 요소의 타입을 제네릭으로 받아, 해당 요소의 Props를 함께 결정하는 구조를 구현하고 싶었기 때문이다.

---

## 3. 제네릭을 이용한 다형성 컴포넌트 구현

### 3.1 가장 단순한 구현

먼저 타입을 고려하지 않으면 다형성 컴포넌트의 구현은 간단하다.

```tsx
function View({
  as: Component = 'div',
  ...props
}: any) {
  return <Component {...props} />;
}
```

`as`가 전달되면 해당 요소를 렌더링하고, 전달되지 않으면 기본적으로 `<div>` 요소를 렌더링한다.

```tsx
<View as="a" href="/books">
  책 목록 보기
</View>
```

하지만 `any`를 사용하면 잘못된 Props를 전달해도 TypeScript가 검사하지 못한다.

```tsx
<View as="button" href="/books">
  잘못된 Props
</View>
```

우리가 원하는 것은 단순히 HTML 요소를 변경하는 기능이 아니다.

**렌더링할 요소가 변경되면 해당 요소에서 사용할 수 있는 Props도 함께 변경되는 컴포넌트**다.

### 3.2 ElementType으로 렌더링할 요소 표현하기

React는 렌더링 가능한 요소의 타입을 표현하기 위해 `ElementType`을 제공한다.

```tsx
import type { ElementType } from 'react';
```

`ElementType`은 `'button'`, `'a'`, `'div'`와 같은 HTML 요소 이름뿐 아니라 React 컴포넌트도 표현할 수 있다.

따라서 `as` 속성에 사용할 수 있다.

```tsx
type Props<T extends ElementType> = {
  as?: T;
};
```

여기서 `T`는 렌더링할 요소의 타입을 의미한다.

이제 `ComponentPropsWithoutRef<T>`를 결합하면 된다.

```tsx
type Props<T extends ElementType> = {
  as?: T;
} & React.ComponentPropsWithoutRef<T>;
```

`as="a"`라면 `T`는 `'a'`로 추론되고, `ComponentPropsWithoutRef<'a'>`를 통해 링크의 Props를 사용할 수 있다.

반대로 `as="button"`이라면 버튼의 Props를 사용할 수 있다.

이렇게 제네릭을 이용해 `as`와 렌더링 대상의 Props를 연결했다.

### 3.3 실제 View 컴포넌트 구현

실제 `View` 구현에서 요소 선택과 Props 전달에 해당하는 부분만 남기면 다음과 같다.

```tsx
export const View = <T extends ElementType>(props: Props<T>) => {
  const { as, sx, style, ...restProps } = props;
  const Component = as || 'div';

  return <Component style={{ ...resolveSx({ sx }), ...style }} {...restProps} />;
};
```

`View`는 `as`로 렌더링할 대상을 고르고 나머지 Props를 전달한다. 프로젝트의 간격 설정인 `sx`는 `resolveSx`로 인라인 스타일로 변환한다. `style`을 뒤에 병합하므로 같은 스타일 속성이 있으면 호출부에서 넘긴 값이 적용된다.

실제 `Props<T>`는 3.2절의 타입에 프로젝트의 `SpacingType`을 더해 `sx`를 받는다.

`T`에는 기본 타입이 지정되어 있지 않다. 따라서 런타임에 `<div>`를 기본으로 렌더링한다는 사실만으로 TypeScript가 `as`를 생략한 경우의 Props를 항상 `<div>` 기준으로 제한한다고 볼 수는 없다.

기본 렌더링 요소와 기본 제네릭 타입은 별도로 관리해야 한다.

---

## 4. 공통 Props와 요소별 Props가 충돌하는 문제

### 4.1 같은 이름을 가진 Props

View를 기반으로 Button을 만들면서 새로운 요구사항이 생겼다.

`Button`은 디자인 크기를 나타내는 `size`를 문자열로 정의한다. 그런데 렌더링 대상인 `<input>`에도 같은 이름의 속성이 있다. `<input>`의 `size`는 숫자다.

```tsx
<input size={20} />
```

이 두 타입을 단순히 교차 타입으로 결합하면 어떻게 될까?

```tsx
type Props =
  { size?: 'small' | 'medium' | 'large' }
  & React.ComponentPropsWithoutRef<'input'>;
```

교차 타입의 `size`는 문자열과 숫자 조건을 동시에 만족해야 한다. 두 조건을 만족하는 크기 값이 없으므로 `size="small"`도 `size={20}`도 전달할 수 없다. 이 경우에는 Button에서 정의한 디자인 크기를 사용하고 싶었다.

### 4.2 Omit으로 중복 Props 제거하기

이 문제를 해결하기 위해 `Omit`을 사용했다.

현재 구현한 타입은 다음과 같다.

```tsx
export type PolymorphicProps<
  T extends ElementType,
  P = {}
> = {
  as?: T;
} & Omit<
  ComponentPropsWithoutRef<T>,
  keyof P
> &
  P &
  SpacingType;
```

`keyof P`는 컴포넌트가 직접 정의한 Prop 이름을 모은다. 예를 들어 `P`에 `size`와 `variant`가 있으면 `keyof P`는 `'size' | 'variant'`다. `Omit`으로 대상 요소의 Props에서 이 이름들을 제외하고 `P`를 결합하면, `size`에는 Button이 정의한 문자열 타입만 적용된다.

### 4.3 현재 PolymorphicProps에서 확인할 부분

현재 구현에서는 렌더링 대상의 Props에서 `keyof P`만 제거한다.

```tsx
Omit<ComponentPropsWithoutRef<T>, keyof P>
```

하지만 `as`와 `SpacingType`도 공통 컴포넌트가 직접 정의하는 Props다.

따라서 이들 역시 렌더링 대상의 Props와 이름이 충돌할 가능성을 고려해야 한다.

이를 명시적으로 표현하면 다음과 같은 형태를 생각할 수 있다.

```tsx
type PolymorphicProps<
  T extends ElementType,
  P = {}
> = {
  as?: T;
} & P &
  SpacingType &
  Omit<
    ComponentPropsWithoutRef<T>,
    keyof P | keyof SpacingType | 'as'
  >;
```

이 방식에서 `Omit`은 이름이 겹칠 때 어떤 Props 정의를 적용할지 결정한다. 실제로 제외할 키는 `SpacingType`을 비롯해 공통 컴포넌트가 직접 관리하는 Props의 범위에 맞춰 정하면 된다.

---

## 5. View를 기반으로 Button 확장하기

### 5.1 Button이 View를 재사용하는 구조

현재 Button은 직접 HTML 요소를 렌더링하지 않고 View를 사용한다.

`Button`은 `shape`, `variant`, `size` 같은 고유 Props로 CSS Module 클래스 이름을 만들고, `leading`과 `trailing`으로 콘텐츠 앞뒤를 구성한다. 렌더링할 요소의 선택과 공통 스타일 처리는 `View`에 맡긴다.

이렇게 역할을 나누고 나니 타입 문제도 두 지점으로 나뉘었다. `as`를 생략했을 때 실제로 렌더링하는 요소와 호출부에서 적용하는 기본 Props 타입이 일치해야 한다. 또 Button이 디자인용 Props를 꺼내고 남긴 값을 View에 전달할 수 있어야 한다.

### 5.2 기본 HTML 요소와 제네릭 기본 타입의 불일치

현재 Button의 Props 타입은 다음과 같이 정의되어 있다.

```tsx
export type AS = 'div';

export type Props<T extends ElementType = AS> =
  PolymorphicProps<T, OwnProps>;
```

`T`의 기본 타입은 `'div'`다.

그런데 실제 Button에서는 다음과 같이 기본 요소를 지정하고 있다.

```tsx
const { as = 'button' } = props;
```

즉, 타입에서는 `'div'`를 기본값으로 정의했지만 실제 렌더링에서는 `'button'`을 기본값으로 사용한다.

여기서 주의해야 할 점이 있다.

`Props<T>`에 기본 제네릭 타입을 정의했다고 해서 Button 함수의 제네릭 타입에도 자동으로 적용되는 것은 아니다.

```tsx
export const Button = <T extends ElementType>(
  props: Props<T>
) => {
  // ...
};
```

Button 함수의 `T`에는 기본 타입이 없다. 따라서 `as`를 생략했을 때 항상 `'div'`나 `'button'`으로 타입이 결정된다고 보장하기 어렵다.

이를 개선하려면 컴포넌트가 실제로 사용하는 기본 요소와 제네릭 기본 타입을 일치시켜야 한다.

```tsx
export type Props<
  T extends ElementType = 'button'
> = PolymorphicProps<T, OwnProps>;

export const Button = <
  T extends ElementType = 'button'
>(
  props: Props<T>
) => {
  // ...
};
```

`as`가 없고 다른 인수로도 `T`를 추론할 수 없으면 기본 타입으로 `'button'`을 사용한다.

### 5.3 제네릭 타입 T와 기본값 'button'의 관계

`as`의 기본값을 적용한 결과에는 호출부에서 선택한 `T`와 런타임 기본값인 `'button'`이 모두 포함될 수 있다. 예를 들어 `T`가 `'a'`로 정해져도, 현재 `as?: T`는 `as`의 생략을 허용한다. 이때 실제 렌더링 대상은 `'button'`이다.

여기까지 확인한 것은 제네릭 기본 타입이 호출부의 추론에 적용되고, 구조 분해 기본값은 함수 내부에서 동작한다는 점이다. 다음으로 확인할 부분은 기본값이 적용된 `as`와 구조 분해 후 남은 Props를 View에 함께 전달할 때 타입 관계가 유지되는지다.

### 5.4 Button에서 View로 Props를 전달하는 문제

Button은 디자인과 콘텐츠 구성에 쓰는 Props를 구조 분해로 꺼낸 뒤, `as`와 나머지 Props를 View에 넘긴다. 전달 지점만 간추리면 다음과 같다.

```tsx
<View as={as} className={classname} style={customStyles} {...restProps}>
  {/* leading, children, trailing으로 콘텐츠를 구성한다. */}
</View>
```

Button의 `PolymorphicProps<T, OwnProps>`는 `OwnProps`의 키를 대상 요소의 Props에서 제외한다. 반면 View는 `ComponentPropsWithoutRef<T>`를 그대로 결합한다. 예를 들어 Button이 디자인용 `size`를 꺼내면 `restProps`에는 그 값이 남지 않는다. 대상이 `<input>`이라면 View에 정의된 숫자 타입의 `size`를 Button의 문자열 값으로 전달해서는 안 된다.

먼저 Button이 소비하는 Props와 View로 전달하는 Props의 경계를 정해야 한다. 그다음 구조 분해 후의 `restProps`가 View에서 선택한 대상의 Props에 할당 가능한지 확인할 수 있다. View에도 같은 `PolymorphicProps`를 적용하는 방안을 생각해 볼 수 있지만, 실제 진단 위치와 할당 가능성은 프로젝트 코드에서 확인해야 한다.

---

## 6. 마무리

다형성 컴포넌트의 타입은 호출부에서 확인할 수 있다. 이 글의 버튼 예시라면, `as="a"`를 지정한 호출은 `href`를 받을 수 있어야 하고, `as`를 생략해 `<button>`을 렌더링하는 호출은 `type="submit"`을 허용하고 `href`는 거부해야 한다. **실제로 렌더링하는 요소와 타입이 허용하는 Props가 일치하는가**가 구현을 검증하는 기준이다.

현재 구현에서는 기본 요소를 생략한 호출과 Button에서 View로 Props를 전달하는 지점을 이 기준으로 더 검증해야 한다. 또한 `ComponentPropsWithoutRef`를 사용하므로 요소별 ref는 지원 범위 밖이다. 링크에 `href`를 필수로 요구하려면 해당 제약도 타입에 별도로 표현해야 한다.
앞으로는 실제 타입 오류를 재현하는 테스트를 작성하고, View를 기반으로 확장한 컴포넌트에서도 타입 안정성이 유지되는지 검증해 보고자 한다.