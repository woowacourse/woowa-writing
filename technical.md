# Blocking VS Non-Blocking, Sync VS Async

오늘은 프로그래밍에서 자주 언급되지만 항상 어려운 개념 중 하나인 Blocking, Non-Blocking 과 Sync, Async 에 대해 이야기해 보려고 합니다. 이 개념들을 왜 공부해야 하는지, 그리고 이들이 우리의 코드와 애플리케이션에 어떤 영향을 미치는지 한번 알아보겠습니다.

## 왜 공부 해야 할까?

### 1. 성능 최적화를 위한 핵심 지식

먼저, 이 개념들은 애플리케이션의 성능을 최적화하는 데 있어 핵심적인 역할을 할 수 있습니다. Blocking 연산은 다른 작업의 실행을 막아 전체적인 성능 저하를 일으킬 수 있습니다. 반면, Non-Blocking과 Async 방식을 적절히 활용하면 리소스를 효율적으로 사용하고 동시에 여러 작업을 처리할 수 있어 전반적인 성능 향상을 기대할 수 있습니다.

### 2. 확장성 있는 시스템 설계

대규모 시스템을 설계할 때, 이 개념들은 더욱 중요해집니다. Blocking 연산이 많은 시스템은 사용자가 증가함에 따라 심각한 성능 저하를 겪을 수 있습니다. Non-Blocking과 Async 패턴을 적절히 활용하면 더 많은 동시 사용자를 처리할 수 있는 확장성 있는 시스템을 구축할 수 있습니다.

---

## Blocking VS Non-Blocking

Blocking 과 Non-Blocking 의 가장 큰 차이는 "제어권"이라는 키워드라고 생각하면 좋을 것 같습니다.
여기서 말하는 제어권이란 `프로그램을 실행할 수 있는 권리`입니다.

### Blocking: 제어권을 넘겨주고 기다리는 방식

Blocking 방식에서는 어떤 작업을 요청했을 때, 그 작업이 완료될 때까지 제어권을 넘겨줍니다. 쉽게 말해, 호출한 함수가 작업을 마칠 때까지 기다리는 거죠.

![image](https://github.com/user-attachments/assets/f839d925-dcff-441b-a67b-5f92e28c9ae5)

이러한 흐름도를 갖습니다. A가 B를 호출 했을 때 A는 B에게 자신의 제어권을 넘겨주게 됩니다. 그리고 B가 실행할 때까지 대기합니다. B가 끝나고 나서야 제어권을 돌려받고 자기의 할 일을 합니다.

코드로 예를 들어볼까요? 
```python
print("파일을 읽기 시작합니다.")
data = file.read()  # Blocking 작업
print("파일 읽기가 끝났습니다.")
```

여기서 `file.read()`는 Blocking 작업입니다. 이 라인에서 프로그램의 실행이 '멈추고', 파일을 다 읽을 때까지 기다립니다. 즉, 제어권을 `file.read()` 함수에 넘겨주고, 그 함수가 작업을 마칠 때까지 다음 줄로 넘어가지 않습니다.

### Non-Blocking: "제어권을 유지하며 다른 작업을 할 수 있는 방식"

반면 Non-Blocking 방식에서는 작업을 요청한 후에도 제어권을 바로 돌려받습니다. 따라서 요청한 작업의 완료 여부와 관계없이 다음 작업을 진행할 수 있죠.

![image](https://github.com/user-attachments/assets/ab632b8f-d2d0-4e36-a302-2d02cb2938c0)

다음과 같은 흐름으로 이루어 집니다. A가 B를 호출하기 위해 제어권을 넘기긴 하지만 바로 돌려받습니다. 그 이후 자신의 다른 일을 할 수 있습니다.

코드로 Non-Blocking 방식의 예를 볼까요?
```python
print("파일을 읽기 시작합니다.")
future = executor.submit(file.read)  # Non-Blocking 작업
print("파일을 읽는 동안 다른 작업을 할 수 있습니다.")
data = future.result()  # 결과가 필요할 때 기다림
print("파일 읽기가 끝났습니다.")
```

여기서 `executor.submit(file.read)`는 Non-Blocking 작업입니다. 파일을 읽는 작업을 시작하지만, 바로 제어권을 돌려받아 다음 줄을 실행할 수 있습니다. 나중에 `future.result()`를 호출할 때 결과가 필요하면 그때 기다리게 되죠.

## Sync VS Async

이번에는 Sync (동기) 와 Async (비동기) 방식에 대해서 알아보겠습니다. 흔히 동기라고 부르는 Synchronous라는 단어를 살펴보면 Syn(함께) + chronous(시간) 두 단어의 합성어 입니다. 
직역하면 **시간을 함께하다** 라는 의미를 가지고 있습니다. 여기서 중요한 것은 **어떤 시간을 함께하냐** 일 것 같습니다.
사람마다 이 **시간** 이라는 개념을 해석하는 기준은 다르겠지만 가장 이해하기 쉬운 기준을 잡아 해석해 보겠습니다.
#### 작업의 종료 == 결괏값의 전달
이 두 가지 시간이 일치할 때 Synchonous, 동기라고 부릅니다.

### Sync (동기): "시간이 일치하는 방식"

Sync 방식에서는 작업의 종료와 결과의 전달이 '동시에' 일어납니다. 즉, 작업을 요청한 주체는 작업을 수행하는 주체의 결과를 받기 위해 기다립니다. 작업이 종료 되자마자 결과를 받아야 하기 때문이죠.

![image](https://github.com/user-attachments/assets/4c119a7b-6558-4cdb-a86c-10db49cf21d9)

때문에 위와 같은 흐름도를 갖습니다. A는 B를 호출합니다. 이후 B가 끝나자마자 결과를 가져와야 하므로 즉, B의 결과가 중요하기 때문에 기다리게 됩니다.

코드로 예를 들어볼까요?
```python
print("커피를 주문합니다.")
coffee = make_coffee()  # Sync 작업
print("커피가 준비되었습니다:", coffee)
```

이 예제에서 `make_coffee()` 함수는 Sync 방식으로 동작합니다. 커피를 주문하는 순간부터 커피가 준비될 때까지 우리는 그 자리에서 기다립니다. 바리스타가 커피를 다 만들었을 때(작업의 종료)와 커피의 전달(결과의 전달)의 '시간'이 일치하는 거죠.

### Async (비동기): "시간이 일치하지 않는 방식"

반면 Async 방식에서는 작업의 종료와 결과의 전달이 일치하지 않습니다. 작업을 요청한 주체는 작업의 완료를 기다리지 않고 다른 일을 할 수 있습니다.

![image](https://github.com/user-attachments/assets/24543513-c339-4f80-8ece-c2a4768eaf3a)


Async 방식의 예를 볼까요?
```python
print("커피를 주문합니다.")
coffee_future = async_make_coffee()  # Async 작업
print(
# ... 다른" >print("커피를 기다리는 동안 다른 일을 합니다.")
# ... 다른 작업 수행 ...
coffee = await coffee_future  # 커피가 준비되면 받아옴
print("커피가 준비되었습니다:", coffee)
```

여기서 `async_make_coffee()`는 Async 방식으로 동작합니다. 커피를 주문한 후 바로 다른 일을 할 수 있습니다. 커피가 언제 준비될지는 모르지만, 준비되면 그때 받아올 수 있죠. 커피를 다 만들었을 때와 커피를 받는 시점의 '시간'이 일치하지 않습니다.

## 어? 뭔가 비슷한데?
여기까지 글을 읽으셨다면 어쨌든 Blocking 과 Sync, Non-Blocking 과 Async 의 흐름이 비슷하다고 느꼈을 것 같습니다.

맞습니다. 사실 동작의 흐름에 있어서는 거의 같은 흐름을 가져가게 됩니다. 이 두 가지의 개념들은 명확한 개념이 아닙니다. `이 코드는 Block 방식이니까 Sync가 아니야!` 라고 말할 수 없습니다. 읽는 사람의 관점에 따라서 block 이 될 수도 있고 sync가 될 수도 있습니다. 혹은 block, sync 둘 다 될 수도 있습니다.


이 두 가지를 가르는 차이는 `관점의 차이`입니다.
같은 코드를 봐도 **제어권이 누구에게 있지?** 로 보게 되면 Block, Non-Block 이 될 수 있습니다.
반면 **시간이 일치하나?** 라는 관점에서 보게 되면 Sync, Async 로 볼 수 있습니다. 

## Blocking, Non-Blocking, Sync, Async 의 조합

![image](https://github.com/user-attachments/assets/499d0472-fbc9-4bdb-8ba3-4d929cebf151)

이 개념들에 대해 공부하다 보면 다음과 같은 표를 보셨을 것 입니다. 마지막으로 이 4가지 조합들이 어떤 흐름을 가지고 어디에 사용되는지 알아보겠습니다.

### Blocking, Sync 조합
![image](https://github.com/user-attachments/assets/1b924ef9-7a89-4570-b987-186cccfb2a4d)


blocking 방식은 제어권을 상대에게 넘기고 대기합니다. Sync 는 요청한 작업이 끝나자마자 결과를 가져와야 하기 때문에 기다립니다.

이런 대표적인 예시로는 IO 작업이 있습니다.

![](https://lh7-rt.googleusercontent.com/slidesz/AGV_vUeZIEc5p6Ur7SqfhCfhZI0nRH50b89cJShgdw2sQbfHoo3uWdXQsZSLcSREuZkj18BdrUn7ntAyBtms0e0P75DYY1s1HuyheT54d2CKiZAtYdO1i0zhRvdMqfk4aesC3mg8ptRFn4R2ZM4DZ-fG9zg5XlRhnCTt=s2048?key=lOUDl-UOaSnRC3izMCOpfw)

위와 같은 코드를 실행하게 되면

![](https://lh7-rt.googleusercontent.com/slidesz/AGV_vUdykr8Qgn9Fp7Bm5vlmYUsYL77KwBn0FCEOCMzHwFUBbMRghiawOoMagKLESBl7Rq0RZrRN5IxMs02ms1jQZJXEeQ7rJWvZlsPCkc_Qp8fBoYmCbx53RAOIb8x59etmDHPy9hj3Gl6Q6GAZTWMoidmLAEDcUY5x=s2048?key=lOUDl-UOaSnRC3izMCOpfw)

이런 콘솔 창이 뜨게 됩니다. 이때 사용자의 입력을 기다리며 대기하게 됩니다.


### Non-Blocking, Sync 조합
![image](https://github.com/user-attachments/assets/3c1412ce-5839-4091-92a8-ca51ee4b3a64)


Non-Blocking 방식은 제어권을 상대에게 넘기지 않고 자신의 작업을 합니다. Sync 는 요청한 작업이 끝나자마자 결과를 가져와야 합니다. 때문에 다른 작업을 하면서도 계속 끝났는지 확인합니다.

이런 예시로는 게임 로딩 창이 있습니다.

![image](https://github.com/user-attachments/assets/53a46a4e-7b92-49ef-bef1-4c8ae9e5766a)


이런 로딩 창을 보신 적 있으실 겁니다. 이 화면에서 챔피언 그림을 누르면 잘 동작이 됩니다(멈춰 있는 것이 아님). 하지만 밑의 로딩 창의 결과가 중요하기 때문에 계속 얼마나 로딩이 되었는지의 결과를 가져옵니다.

### Blocking, Async 조합

![image](https://github.com/user-attachments/assets/4605d45f-072f-4d70-b66c-91ebe7e3403d)


blocking 방식은 제어권을 상대에게 넘기고 대기합니다. Async 는 요청한 작업이 끝나자마자 결과를 가져와야 할 필요가 없습니다. 작업의 종료가 중요하지 않지만 기다려야 합니다.

그림만 봐도 효율적이지 않은 조합이라고 느껴집니다. 실제로 Blocking, Async 조합은 안티패턴 이라고 불리기도 합니다.


### Non-Blocking, Async 조합
![image](https://github.com/user-attachments/assets/61901734-bbdd-4465-9d23-7515b03a074f)


Non-Blocking 방식은 제어권을 상대에게 넘기지 않고 자신의 작업을 합니다. Async 는 요청한 작업이 끝나자마자 결과를 가져와야 할 필요가 없습니다. 때문에 위와 같은 흐름도를 가지게 됩니다.

보통 JavaScript 의 Ajax 요청이 이런 방법을 사용합니다.
```javascript
console.log("데이터 요청 시작");

fetch('https://api.example.com/data')
  .then(response => response.json())
  .then(data => {
    console.log("받은 데이터:", data);
  })
  .catch(error => {
    console.error("에러 발생:", error);
  });

console.log("다음 작업 실행");
```


## 마무리 하며
Blocking, Non-Blocking, Sync, Async 에 대해 알아봤습니다.
이 개념들은 최적화를 위해 자주 등장하는 개념이기 때문에 이번 기회에 알아가셨으면 좋겠습니다!
