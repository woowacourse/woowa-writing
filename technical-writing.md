# 리액트 신기능 Activity 적용기

리액트가 19.2 버전으로 업데이트되며 새로운 기능이 추가되었습니다. 그중 가장 제 눈에 들어온 추가기능은 `<Activity />` 였습니다.

Activity는 서비스를 “활동 여부”에 따른 여러 요소로 쪼개는 기능입니다. 활동을 하는 요소만 보이고 활동을 하고 있지 않은 요소는 숨기는 기능이죠. 이번 글에서는 이 Activity를 이용해 프로젝트 내 사용자 입력 컴포넌트를 개선한 경험을 공유하고자 합니다.

<br />

## 예상 독자

- 리액트 19.2 버전의 신 기능인 Activity에 대해 알아보고 싶은 분들
- 조건에 따라 표시되는 UI를 리액트로 만들어보고 싶은 분들
  - 해당 UI를 만들었는데, 새롭게 표시할 때마다 기존값이 초기화되어 어려움을 겪고 있는 분들
- 내 프로젝트에 어떻게 Activity를 녹여낼 수 있을까 고민하고 계신 분들

<br />

## Activity란?

**Activity는 조건에 따라 특정 UI의 표시 여부를 다루는 리액트 컴포넌트**입니다.

웹서비스에서 특정 조건에만 보이는 UI는 굉장히 흔합니다. 메뉴 버튼을 눌렀을 때 보이는 사이드바, 특정 버튼을 눌렀을 때 나오는 모달 창, 다음 버튼을 눌렀을 때 넘어가는 화면 등 다양한 요소가 있죠.

간단한 모달 창을 예시로 들어보겠습니다. 모달 창의 기능을 나열하면 다음과 같습니다.

- 모달 열기를 누르면 모달 창이 표시된다.
- 모달 내부에 있는 모달 닫기를 누르면 모달 창이 사라진다.

![ModalExample.gif](https://github.com/MinSungJe/woowa-writing/blob/level4/images/ModalExample.gif)

지금까지 리액트로 이런 모달을 구현하기 위해 보통 && 연산자를 사용했습니다.

```tsx
function App() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div className={S.Wrapper}>
      <button
        type="button"
        onClick={() => {
          setIsOpen(true);
        }}>
        모달 열기
      </button>
      // && 연산자를 이용해 조건부 표시
      {isOpen && (
        <Modal
          onClose={() => {
            setIsOpen(false);
          }}
        />
      )}
    </div>
  );
}

export default App;
```

하지만 이번 19.2 업데이트로 추가된 Activity를 이용해서도 해당 모달을 만들 수 있습니다.

```tsx
import { Activity, useState } from 'react';

function App() {
  const [isOpen, setIsOpen] = useState(false);

  return (
    <div className={S.Wrapper}>
      <button
        type="button"
        onClick={() => {
          setIsOpen(true);
        }}>
        모달 열기
      </button>
      // Activity를 이용해 조건부 표시
      <Activity mode={isOpen ? 'visible' : 'hidden'}>
        <Modal
          onClose={() => {
            setIsOpen(false);
          }}
        />
      </Activity>
    </div>
  );
}

export default App;
```

그렇다면 여기서 한 가지 궁금증이 생깁니다. 지금까지는 && 연산자만으로 구현할 수 있었던 기능인데, 왜 새롭게 Activity를 추가했을까요? Activity로 구현한 요소의 특징을 알아보도록 하겠습니다.

<br />

## Activity의 특징

### 요소를 없애지 않는다. 요소를 숨길 뿐이다.

Activity의 가장 큰 특징은 요소의 DOM 처리에 있습니다.

Activity의 경우 DOM에는 있지만 style 속성을 이용해 요소를 숨깁니다. 개발자 도구를 살펴보면 DOM에는 해당 요소가 있지만 `display: none;`을 주어 숨기고 있음을 알 수 있습니다.

![activity.gif](https://github.com/MinSungJe/woowa-writing/blob/level4/images/activity.gif)

반면 && 연산자의 경우 DOM에서 요소를 삭제합니다. 따라서 특정 조건일 때 해당 요소가 보이지 않습니다.

![andand.gif](https://github.com/MinSungJe/woowa-writing/blob/level4/images/andand.gif)

정리하자면 Activity를 이용한 모달은 DOM 요소에는 있지만 스타일을 통해 숨기거나 보입니다. 반면 &&를 이용한 모달은 DOM 요소에서 아예 없어졌다가 다시 생깁니다. 이 차이점으로 어떤 이점을 가져올 수 있을까요?

### 가지고 있는 상태 값을 유지한다.

가장 큰 장점은 모달 내부의 상태를 유지할 수 있다는 점입니다. 모달 자체를 언마운트하지 않고 숨기기만 하기 때문이죠. useState 등으로 선언한 상태뿐만 아니라 input 내부에 일시적으로 입력한 값도 유지할 수 있습니다.

![inputSustain](https://github.com/MinSungJe/woowa-writing/blob/level4/images/inputSustained.gif)

기존 &&로 구현한 모달의 경우 값이 유지되지 않습니다.

![noInputSustain](https://github.com/MinSungJe/woowa-writing/blob/level4/images/noInputSustain.gif)

### 미리 요소를 렌더링할 수 있다.

또한 모달 내부 요소를 사전에 렌더링할 수 있습니다. 리액트는 Activity의 mode가 hidden인 경우에도 자식 요소가 낮은 우선순위로 계속 렌더링합니다. 이러한 사전 렌더링 덕분에 코드나 데이터를 미리 로드할 수 있어 성능상으로 이점을 가져올 수 있습니다.

이뿐만 아니라 요소 내부에서 useEffect 등으로 선언한 Effects는 자체적으로 정리합니다. 이를 통해 불필요한 사이드 이펙트를 방지할 수 있죠.

<br />

## 포게더 프로젝트에 적용

지금까지 Activity의 장점에 대해 알아보았습니다. 그러면 Activity를 프로젝트에 어떻게 녹여낼 수 있을까요?

### 기존 버전

제 프로젝트에서 조건부로 바뀌는 UI를 살펴보니 사용자의 입력을 받는 페이지가 있었습니다.

포게더 서비스에선 사용자의 입력을 여러 페이지를 통해 받습니다. 한 곳에 너무 많은 입력이 있을 경우 사용자는 부담을 가질 수 있습니다. 따라서 여러 단계로 이루어진 페이지로 입력을 분산했습니다.

![funnel.png](https://github.com/MinSungJe/woowa-writing/blob/level4/images/funnel.png)

페이지를 분산할 때 라우터를 따로 둘 수도 있지만 저는 하나의 라우터로 모든 페이지를 관리하고 싶었습니다. 웹의 특성상 사용자는 url을 마음대로 입력할 수 있습니다. 만약 페이지별로 라우터를 따로 둔다면 사용자가 url을 입력해서 중간 단계에 접속할 수 있습니다. 이는 예상하지 못한 동작을 유발할 수 있겠죠.

포게더는 하나의 라우터 안에 단계별로 페이지를 다르게 보여주는 요소를 구현했습니다. 그리고 이 구현을 위해 && 연산자를 사용했습니다.

```tsx
const [step, setStep] = useState('name');

{step === 'name' && (
  <SpaceNameElement ... />
)}
{step === 'accessType' && (
  <SpaceVisibilityElement ... />
)}
{step === 'description' && (
  <SpaceDescriptionElement ... />
)}
{step === 'detail' && (
  <SpaceDetailElement ... />
)}
{step === 'check' && (
  <SpaceCheckElement ... />
)}
```

### 어떤 문제점이 발생했을까?

하지만 이 요소는 문제가 발생했는데요. 각 페이지를 왔다 갔다 할 때 입력값이 초기화되는 현상이 있었습니다.

![resetInput.gif](https://github.com/MinSungJe/woowa-writing/blob/level4/images/resetInput.gif)

이유는 앞서 언급한 && 연산자로 구현한 요소의 문제점과 같습니다. &&로 요소를 숨길 경우 요소 자체가 DOM에서 언마운트되기에 상태를 유지하지 않습니다. 따라서 입력값이 초기화 되는 것이죠.

그렇기에 Activity가 나오기 전에는 초기 표시 값을 props로 전달하여 문제를 해결했습니다. 단계를 거치며 입력한 값을 저장하는 상태를 선언하고, 저장된 값을 초기 렌더링을 할 때 표시하도록 추가로 구현했죠.

```tsx
const [step, setStep] = useState('name');

// 단계를 거치며 입력된 값 상태
const [form, setForm] = useState({
  name: '',
  visibility: 'PUBLIC',
  description: '',
  profileImage: [],
  email: '',
  instagram: '',
});

{step === 'name' && (
  <SpaceNameElement initialValue={form.name} ... />
)}
{step === 'accessType' && (
  <SpaceVisibilityElement initialValue={form.visibility} ... />
)}
{step === 'description' && (
  <SpaceDescriptionElement initialValue={form.description} ... />
)}
{step === 'detail' && (
  <SpaceDetailElement initialValue={{profileImage: form.profileImage ...}} ... />
)}
{step === 'check' && (
  <SpaceCheckElement ... />
)}
```

이처럼 지금까지는 초기값의 유지를 위해 추가적인 구현이 필요했습니다.

텍스트의 경우 초기값을 전달하면 그것을 상태 값에 담아주기만 하면 되기에, 그렇게 어려운 로직은 아니었습니다.

하지만 사진의 경우 꽤 애를 먹었는데요. 미리보기 사진 이미지를 상태에 따로 담고 초기에 표시하는 로직을 구현할 때 어려움을 겪었습니다. 또한 제출하는 상태 이외에 미리보기를 위한 다른 상태를 선언한 것이기에 불필요한 재렌더링이 발생합니다.

추가적인 구현이 필요할 뿐만 아니라 성능적으로도 바람직하지 못한 로직인 셈입니다.

### Activity로 개선

이제 위 코드에 Activity를 적용해 보겠습니다. 작성한 && 연산자를 지우고 Activity로 대체합니다. 그리고 속성으로 mode를 계산하여 전달합니다.

```tsx
<Activity mode={Funnel.funnelStep === 'name' ? 'visible' : 'hidden'}>
  <SpaceNameElement ... />
</Activity>
<Activity mode={Funnel.funnelStep === 'accessType' ? 'visible' : 'hidden'}>
  <SpaceVisibilityElement ... />
</Activity>
<Activity mode={Funnel.funnelStep === 'description' ? 'visible' : 'hidden'}>
  <SpaceDescriptionElement ... />
</Activity>
<Activity mode={Funnel.funnelStep === 'detail' ? 'visible' : 'hidden'}>
  <SpaceDetailElement ... />
</Activity>
<Activity mode={Funnel.funnelStep === 'check' ? 'visible' : 'hidden'}>
  <SpaceCheckElement ... />
</Activity>
```

이제 별다른 추가 로직 없이 사용자가 입력했던 값을 유지합니다.

![unresetInput.gif](https://github.com/MinSungJe/woowa-writing/blob/level4/images/unresetInput.gif)

Activity는 요소를 아예 언마운트시켰다가 다시 만드는 것이 아닙니다. CSS 속성을 이용해 잠시 숨기는 것입니다. 따라서 상태 값이 없어지지 않기에 초기값을 위한 추가 로직을 구현할 필요가 없습니다.

DOM 구조를 확인하면 CSS 속성을 이용해 잠시 숨기는 동작을 더욱 명확하게 볼 수 있는데요. 표시되는 단계가 바뀌면 DOM엔 요소가 그대로 있지만 style만 변하는 것을 알 수 있습니다.

![steps.gif](https://github.com/MinSungJe/woowa-writing/blob/level4/images/steps.gif)

<br />

## 마치며

리액트의 19.2 업데이트로 생긴 Activity는 DOM에서 요소를 없애지 않고 숨기기만 합니다. 따라서 기존에 && 연산자를 이용해 요소를 만들었을 때와 비교해 많은 이점을 가져올 수 있습니다.

포게더 프로젝트에 적용하며 가장 크게 느낀 변화는 상태 관리의 단순화였습니다. Activity 덕분에 이제 초기값을 유지하기 위한 상태는 선언하지 않아도 됩니다. 오로지 입력값을 관리한다는 요소의 본분에 집중할 수 있게 되었죠.

Activity는 조건부 렌더링이 필요한 컴포넌트의 성능과 개발 편의성을 동시에 높여주는 유용한 기능이라는 생각이 들었습니다.

<br />

## 참고

- [React 19.2](https://react.dev/blog/2025/10/01/react-19-2)

- [React - Activity](https://react.dev/reference/react/Activity)
