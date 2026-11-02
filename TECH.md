# 다형성 컴포넌트를 직접 만들며 마주친 TypeScript 타입 문제들

React에서 공통 UI 컴포넌트를 만들다 보면 같은 디자인을 유지하면서 서로 다른 HTML 요소를 렌더링해야 하는 상황을 마주한다.

예를 들어 버튼처럼 생긴 UI를 생각해 보자. 사용자가 폼을 제출할 때는 `<button>` 요소를 사용하지만, 다른 페이지로 이동할 때는 `<a>` 요소를 사용하는 것이 적절하다.

두 요소는 동일한 디자인을 가질 수 있지만 HTML에서 담당하는 역할은 다르다.

처음에는 컴포넌트에 `as`라는 속성을 추가해 렌더링할 HTML 요소를 변경하면 된다고 생각했다. 하지만 TypeScript를 적용하면서 생각보다 많은 부분을 고려해야 했다.

`as`에 전달한 요소에 따라 허용되는 Props가 달라져야 했고, 공통 컴포넌트에서 정의한 Props와 HTML의 기본 Props가 충돌하는 문제도 있었다. 특히 다형성 컴포넌트를 다른 공통 컴포넌트의 기반으로 사용하면서 제네릭 타입을 전달하는 과정이 복잡해졌다.

이 글에서는 공통 UI 컴포넌트인 `View`와 이를 활용하는 `Button`을 구현한 과정을 바탕으로 다형성 컴포넌트의 타입을 설계하는 방법을 살펴본다. 또한 현재 구현에서 확인한 문제와 아직 해결하지 못한 한계도 함께 정리한다.

---

## 1. 공통 컴포넌트에서 HTML 요소가 고정되는 문제

### 1.1 같은 디자인, 다른 HTML 요소

다음과 같은 공통 Button 컴포넌트가 있다고 가정해 보자.

```tsx
type ButtonProps = {
  children: React.ReactNode;
  onClick?: () => void;
};

function Button({ children, onClick }: ButtonProps) {
  return (
    <button className="button" onClick={onClick}>
      {children}
    </button>
  );
}
```

이 컴포넌트는 버튼의 디자인을 재사용할 수 있다는 장점이 있다. 하지만 언제나 `<button>` 요소를 렌더링한다는 제약이 있다.

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

React에는 `ButtonHTMLAttributes`, `AnchorHTMLAttributes`와 같은 타입도 존재한다.

```tsx
type ButtonProps =
  React.ButtonHTMLAttributes<HTMLButtonElement>;
```

이 타입 역시 버튼의 HTML 속성을 표현할 수 있다.

다만 두 타입은 완전히 동일하지 않다.

`ComponentPropsWithoutRef<'button'>`는 JSX에서 해당 요소에 허용되는 Props를 가져오므로, `ButtonHTMLAttributes`보다 일부 속성의 타입이 더 구체적이다.

예를 들어 `type` 속성은 JSX의 `<button>`에서 `'button'`, `'submit'`, `'reset'`으로 제한된다.

이번 구현에서는 HTML 요소뿐 아니라 React 컴포넌트도 렌더링할 수 있도록 만들고 싶었다.

따라서 요소의 타입을 전달받아 Props를 추출할 수 있는 `ComponentPropsWithoutRef`를 사용했다.

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

이렇게 제네릭을 이용해 `as`와 HTML Props를 연결했다.

### 3.3 실제 View 컴포넌트 구현

현재 작성한 View 컴포넌트는 다음과 같다.

```tsx
export const View = <T extends ElementType>(
  props: Props<T>
) => {
  const { as, sx, style, ...restProps } = props;

  const Component = as || 'div';

  const spacingStyle = resolveSx({ sx });
  const customStyles = { ...spacingStyle, ...style };

  return (
    <Component
      style={customStyles}
      {...restProps}
    />
  );
};
```

View는 다음과 같은 역할을 수행한다.

- `as`를 통해 렌더링할 요소를 결정한다.
- `sx`를 통해 공통 간격 스타일을 계산한다.
- `style`을 병합해 최종 스타일을 구성한다.
- 나머지 Props를 실제 HTML 요소에 전달한다.

`as`를 전달하지 않으면 기본적으로 `<div>`를 렌더링한다.

여기서 `sx`는 프로젝트에서 정의한 스타일 관련 Props다. `resolveSx`는 전달된 `sx`를 React의 인라인 스타일 객체로 변환하는 역할을 한다.

View를 기반으로 다른 UI 컴포넌트를 만들면, HTML 요소 선택과 공통 스타일 처리 로직을 재사용할 수 있다.

다만 이 구현에는 아직 고려할 부분이 있다.

`T`에는 기본 타입이 지정되어 있지 않다. 따라서 런타임에 `<div>`를 기본으로 렌더링한다는 사실만으로 TypeScript가 `as`를 생략한 경우의 Props를 항상 `<div>` 기준으로 제한한다고 볼 수는 없다.

기본 렌더링 요소와 기본 제네릭 타입은 별도로 관리해야 한다.

---

## 4. 공통 Props와 HTML Props가 충돌하는 문제

### 4.1 같은 이름을 가진 Props

View를 기반으로 Button을 만들면서 새로운 요구사항이 생겼다.

Button은 HTML 요소의 기본 속성 외에도 디자인을 결정하는 Props가 필요하다.

```tsx
export type OwnProps = {
  shape?: 'default' | 'normal' | 'link';
  variant?:
    | 'default'
    | 'primary'
    | 'secondary'
    | 'accent'
    | 'ghost'
    | 'danger'
    | 'danger-weak';
  size?: 'x-small' | 'small' | 'medium' | 'large';
  align?: 'center' | 'left' | 'right';
  block?: boolean;
  leading?: React.ReactNode;
  trailing?: React.ReactNode;
};
```

문제는 HTML 요소에도 같은 이름의 속성이 존재할 수 있다는 점이다.

대표적인 예가 `size`다.

Button에서는 `size`를 디자인 크기를 나타내는 문자열로 정의했다.

```tsx
size?: 'small' | 'medium' | 'large';
```

하지만 HTML의 `<input>` 요소에서 `size`는 숫자 타입이다.

```tsx
<input size={20} />
```

이 두 타입을 단순히 교차 타입으로 결합하면 어떻게 될까?

```tsx
type Props =
  { size?: 'small' | 'medium' | 'large' }
  & React.ComponentPropsWithoutRef<'input'>;
```

동일한 `size` 속성에 서로 다른 타입이 적용된다.

교차 타입은 한쪽 타입을 다른 타입으로 덮어쓰는 방식이 아니다. 양쪽 조건을 모두 만족해야 한다.

따라서 `size`에 문자열 유니언 타입과 숫자 타입이 동시에 요구되면 유효한 값을 전달하기 어려워진다.

우리가 원하는 것은 Button이 직접 정의한 `size`를 사용하는 것이다.

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

여기서 중요한 부분은 다음 코드다.

```tsx
Omit<ComponentPropsWithoutRef<T>, keyof P>
```

`keyof P`는 `P` 타입이 가진 속성 이름을 유니언 타입으로 추출한다.

예를 들어 다음 타입이 있다면,

```tsx
type OwnProps = {
  size?: 'small' | 'medium' | 'large';
  variant?: 'primary' | 'secondary';
};
```

`keyof OwnProps`는 다음과 같다.

```tsx
'size' | 'variant'
```

따라서 `Omit`은 HTML 요소의 기본 Props에서 `size`와 `variant`를 제거한다.

이후 `P`를 결합하면 컴포넌트에서 직접 정의한 Props를 사용할 수 있다.

즉, HTML의 기본 Props와 공통 컴포넌트의 고유 Props가 충돌하지 않도록 만든 것이다.

### 4.3 현재 PolymorphicProps에서 확인할 부분

현재 구현에서는 HTML Props에서 `keyof P`만 제거한다.

```tsx
Omit<ComponentPropsWithoutRef<T>, keyof P>
```

하지만 `as`와 `SpacingType`도 공통 컴포넌트가 직접 정의하는 Props다.

따라서 이들 역시 HTML Props와 이름이 충돌할 가능성을 고려해야 한다.

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

이렇게 하면 공통 컴포넌트가 직접 관리하는 Props를 HTML Props보다 우선하도록 설계할 수 있다.

다만 실제 프로젝트에서는 `SpacingType`이 어떤 속성을 정의하는지 확인한 뒤 충돌 가능성을 판단해야 한다.

중요한 점은 `Omit`이 단순히 타입 오류를 없애는 도구가 아니라는 것이다.

**어떤 Props의 정의를 우선할 것인지 명시하는 수단**으로 사용할 수 있다.

---
## 5. View를 기반으로 Button 확장하기

### 5.1 Button이 View를 재사용하는 구조

현재 Button은 직접 HTML 요소를 렌더링하지 않고 View를 사용한다.

핵심 구조만 남기면 다음과 같다.

```tsx
export const Button = <T extends ElementType>(
  props: Props<T>
) => {
  const {
    as = 'button',
    shape = 'default',
    variant = 'default',
    size = 'medium',
    ...restProps
  } = props;

  return <View as={as} {...restProps} />;
};
```

실제 구현에서는 `shape`, `variant`, `size`, `align`, `block`을 사용해 CSS Module 클래스 이름을 조합한다. 또한 `leading`, `trailing`을 통해 버튼 콘텐츠 앞뒤에 요소를 배치한다.

이렇게 하면 View는 **HTML 요소 선택과 공통 스타일 처리**를 담당하고, Button은 **버튼 고유의 디자인과 콘텐츠 구성**을 담당하게 된다.

두 컴포넌트의 역할을 분리하면서 중복된 구현을 줄일 수 있다.

하지만 여기서 새로운 타입 문제가 발생할 수 있다.

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

이렇게 하면 `as`를 생략했을 때 기본적으로 버튼의 Props를 사용하도록 타입을 설계할 수 있다.

다만 기본 제네릭 타입을 지정하는 것과 함수 내부의 타입 오류를 해결하는 것은 별개의 문제다.

### 5.3 제네릭 타입 T와 기본값 'button'의 관계

다음 코드를 다시 살펴보자.

```tsx
export const Button = <T extends ElementType>(
  props: Props<T>
) => {
  const { as = 'button' } = props;
};
```

여기서 `as`는 제네릭 타입 `T`와 연결되어 있다.

하지만 `'button'`은 구체적인 문자열 리터럴 타입이다.

문제는 `T extends ElementType`이라는 제약만으로 `T`가 반드시 `'button'`을 허용한다고 보장할 수 없다는 점이다.

예를 들어 `T`가 `'a'`라면 `T`는 `ElementType`의 조건을 만족하지만 `'button'`과는 다른 타입이다.

따라서 TypeScript는 제네릭 타입과 기본값 사이의 관계를 단순하게 판단할 수 없다.

이러한 문제는 다형성 컴포넌트에서 자주 마주치는 부분이다.

제네릭 기본값을 `'button'`으로 설정하더라도 함수 내부에서는 `T`가 다른 요소일 가능성을 고려해야 한다.

현재 코드의 타입 오류를 정확히 해결하려면 실제 TypeScript 진단 메시지와 오류가 발생한 위치를 확인해야 한다.

### 5.4 Button에서 View로 Props를 전달하는 문제

Button의 실제 구현에서는 여러 Props를 구조 분해한 뒤 나머지 Props를 View로 전달한다.

```tsx
const {
  as = 'button',
  shape = 'default',
  variant = 'default',
  size = 'medium',
  align = 'center',
  block,
  children,
  leading,
  trailing,
  className,
  sx,
  style,
  ...restProps
} = props;
```

이후 다음과 같이 View를 렌더링한다.

```tsx
<View
  as={as}
  className={classname}
  style={customStyles}
  {...restProps}
>
  {leading && (
    <span className={styles.leading}>
      {leading}
    </span>
  )}
  {children}
  {trailing && (
    <span className={styles.trailing}>
      {trailing}
    </span>
  )}
</View>
```

여기서 Button과 View는 서로 다른 Props 타입을 사용한다.

Button은 다음 타입을 사용한다.

```tsx
type ButtonProps<T extends ElementType> =
  PolymorphicProps<T, OwnProps>;
```

반면 View는 다음 타입을 사용한다.

```tsx
type ViewProps<T extends ElementType> = {
  as?: T;
} & ComponentPropsWithoutRef<T> &
  SpacingType;
```

Button의 타입은 `OwnProps`에 포함된 속성을 HTML Props에서 제거한 뒤 다시 결합한다.

하지만 View의 타입은 `ComponentPropsWithoutRef<T>`를 그대로 사용한다.

따라서 두 타입은 구조적으로 동일하지 않다.

특히 `size`처럼 Button에서 재정의한 속성은 두 타입 사이에서 서로 다른 의미를 가질 수 있다.

또한 TypeScript가 제네릭 타입 `T`에 대한 `Omit` 결과와 구조 분해 이후의 `restProps`를 View가 요구하는 Props와 호환된다고 증명하지 못할 가능성이 있다.

이 문제를 해결할 때 중요한 것은 단순히 타입 단언을 추가하는 것이 아니다.

**Button에서 제거한 Props와 View에 전달할 Props의 관계를 타입으로 어떻게 표현할 것인지** 먼저 결정해야 한다.

예를 들어 View와 Button이 동일한 다형성 Props 설계 규칙을 사용하도록 정리할 수 있다.

```tsx
type ViewProps<T extends ElementType> =
  PolymorphicProps<T, {}>;
```

다만 이렇게 변경하는 것만으로 Button의 `restProps` 전달 문제가 해결된다고 보장할 수는 없다.

제네릭 타입과 `Omit`을 조합한 객체의 할당 가능성은 별도로 검증해야 한다.

또한 View가 공통 컴포넌트의 기반 역할을 한다면, View가 반드시 알아야 하는 Props와 하위 컴포넌트에서만 사용하는 Props를 명확히 구분하는 것이 중요하다.

---

## 6. 구현을 통해 확인한 결과와 한계

현재 구현에서는 하나의 Button 컴포넌트에서 서로 다른 HTML 요소를 렌더링할 수 있다.

```tsx
<Button type="submit">
  제출하기
</Button>

<Button as="a" href="/books">
  책 목록 보기
</Button>
```

두 컴포넌트는 동일한 Button 스타일을 사용하지만 렌더링되는 HTML 요소는 다르다.

이를 통해 HTML 요소의 의미를 유지하면서 디자인을 재사용할 수 있게 되었다.

또한 `ComponentPropsWithoutRef<T>`를 활용해 HTML 요소의 기본 속성을 직접 정의하지 않고 재사용할 수 있다.

하지만 현재 구현에는 몇 가지 한계가 남아 있다.

첫째, 기본 HTML 요소와 제네릭 기본 타입이 일치하지 않는다. Button은 실제로 `<button>`을 렌더링하지만 Props의 기본 타입은 `'div'`로 정의되어 있다.

둘째, View와 Button이 서로 다른 방식으로 HTML Props를 결합한다. 이 때문에 Button의 Props를 View로 전달할 때 타입 호환성을 추가로 검토해야 한다.

셋째, ref를 지원하지 않는다. 현재 `ComponentPropsWithoutRef`를 사용하므로, 렌더링하는 요소에 따라 달라지는 ref 타입을 외부에 제공하는 구조는 구현되어 있지 않다.

마지막으로 TypeScript가 검사할 수 있는 것은 타입에 표현된 제약뿐이다. 예를 들어 `<a>` 요소에 `href`가 반드시 존재해야 한다는 요구사항을 추가하고 싶다면, 기본 HTML Props 타입을 가져오는 것만으로는 충분하지 않다. 별도의 타입 설계가 필요하다.

따라서 다형성 컴포넌트의 구현을 완료했다고 판단하려면 렌더링 결과뿐 아니라 허용해야 할 Props와 거부해야 할 Props를 각각 검증해야 한다.

---

## 7. 마무리

처음에는 공통 컴포넌트에서 HTML 요소를 변경하는 것이 목표였다.

하지만 실제로 View와 Button을 구현하면서 렌더링할 요소를 변경하는 것보다 해당 요소에 맞는 Props 타입을 연결하는 과정이 더 복잡하다는 것을 알게 되었다.

`ElementType`으로 렌더링할 요소를 표현하고, `ComponentPropsWithoutRef`로 해당 요소의 Props를 가져올 수 있었다.

또한 공통 컴포넌트가 직접 정의한 Props와 HTML Props가 충돌할 때는 `Omit`을 이용해 어떤 속성을 우선할지 결정할 수 있었다.

특히 View를 기반으로 Button을 확장하면서 제네릭 타입을 여러 컴포넌트에 걸쳐 유지하는 문제를 확인했다.

다형성 컴포넌트에서 중요한 것은 단순히 `as`를 통해 원하는 HTML 요소를 렌더링하는 것이 아니다.

**렌더링할 요소와 Props 사이의 관계를 TypeScript가 올바르게 검사할 수 있도록 설계하는 것**이 핵심이다.

현재 구현은 기본적인 다형성 컴포넌트의 구조를 갖추고 있지만, 제네릭 Props 전달과 ref 지원 등 추가로 개선해야 할 부분이 남아 있다.

앞으로는 실제 타입 오류를 재현하는 테스트를 작성하고, View를 기반으로 확장한 컴포넌트에서도 타입 안정성이 유지되는지 검증해 보고자 한다.
