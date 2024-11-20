## 목차

1. 시간 복잡도와 효율적인 알고리즘 구현
2. Big-O 표기법의 이해
3. Big-O 표기법의 종류와 사례
4. 효율적인 탐색 알고리즘: 이진 탐색
5. 탐욕적 접근법: 그리디 알고리즘
6. 최적화 기법: 동적 프로그래밍
7. 코딩 문제 예시 및 해설

# **⏰ Time Complexity (시간 복잡도)**

> **Time Complexity (시간 복잡도)를 고려한 효율적인 알고리즘 구현 방법에 대한 고민과                   Big-O 표기법을 이용해 시간 복잡도를 나타내는 방법에 대해 알아봅시다.**
> 

## 1. 시간 복잡도와 효율적인 알고리즘 구현
알고리즘 문제를 해결할 때는 답을 찾는 것뿐 아니라, 효율적으로 풀었는지도 중요합니다.
"이 방법이 최적인가?", "더 나은 방법이 있을까?"라는 의문을 품고 효율성을 고민하는 것이 시간 복잡도를 줄이는 문제로 이어집니다. 이를 위해 Big-O 표기법을 사용하여 시간 복잡도를 표현합니다.

---

## **❗️시간복잡도**

**• 문제를 해결하기 위한 알고리즘의 로직을 코드로 구현할 때, 시간 복잡도를 고려한다는 것은 무슨 의미일까?**

> **알고리즘의 로직을 코드로 구현할 때, 시간 복잡도를 고려한다는 것은 ‘입력값의 변화에 따라 연산을 실행할 때, 연산 횟수에 비해 시간이 얼마만큼 걸리는가?’라는 말이다.**
> 

**• 효율적인 알고리즘을 구현한다는 것은 바꾸어 말해 입력값이 커짐에 따라 증가하는 시간의 비율을 최소화한 알고리즘을 구성했다는 이야기이다.**

**• 그리고 이 시간 복잡도는 Big-O 표기법을 사용해 나타낸다.**

---

## **❗️Big-O 표기법**

## 시간 복잡도를 표기하는 방법

- **Big-O (빅-오): 상한 점근 (Worst-case)** - `최악`
- **Big-Ω (빅-오메가)**: 하한 점근 (Best-case) - `최선`
- **Big-θ (빅-세타)**: 평균 (Average-case) - `중간`

## 🚀 가장 자주 사용되는 표기법

빅오 표기법은 최악의 경우를 고려하므로, 프로그램이 실행되는 과정에서 소요되는 최악의 시간까지 고려할 수 있습니다.

$$
왜 최악으로 생각을 해야하나 ❓
$$

- **최선의 경우**: 결과를 반환하는 데 최선의 경우 1초, 평균적으로 1분, 최악의 경우 1시간이 걸리는 알고리즘을 구현했다고 가정할 때, 최선의 경우를 고려하면 100번 실행 시 100초가 걸립니다.
- **중간의 경우**: 평균값을 기대하는 시간 복잡도를 고려하면 100번 실행할 때 100분의 시간이 소요되지만, 최악의 경우 몇 번 발생하면 시간이 300분을 넘길 수 있습니다.
- **최악의 경우**: 최악의 경우를 고려하여 시간을 계산하는 것이 바람직합니다.

## 🚀 Big-O 표기법의 종류

다양한 시간 복잡도의 Big-O 표기법을 이해하면 문제를 더 효과적으로 해결할 수 있습니다.

1. **O(1)**: 일정한 복잡도 (constant complexity)
2. **O(n)**: 선형 복잡도 (linear complexity)
3. **O(log n)**: 로그 복잡도 (logarithmic complexity)
4. **O(n^2)**: 2차 복잡도 (quadratic complexity)
5. **O(2^n)**: 기하급수적 복잡도 (exponential complexity)

## 예시

![시간복잡도 그래프](https://github.com/user-attachments/assets/d73886e8-7a5c-43ed-a81d-08d45535421d)

---

### ❗️O(1)

> **O(1)는 일정한 복잡도(constant complexity)라고 하며, 입력값이 증가하더라도 시간이 늘어나지 않는다.**
> 

```python
def O_1_algorithm(arr, index):
    return arr[index]

arr = [1, 2, 3, 4, 5]
index = 1
result = O_1_algorithm(arr, index)
print(result)  # Output: 2

```

![O1](https://github.com/user-attachments/assets/e2d9f565-d499-47a8-abc3-ac1d2e27dd8c)

---

### ❗️**O(n) - Linear Time Complexity**

> **O(n)은 선형 복잡도(linear complexity)라고 부르며, 입력값이 증가함에 따라 시간 또한 `같은 비율`로 증가하는 것을 의미한다.**
> 

```python
def O_n_algorithm(n):
    for i in range(n):
        # 무언가의 행동
        pass

def another_O_n_algorithm(n):
    for i in range(2 * n):
        # 무언가의 행동
        pass
```

![On](https://github.com/user-attachments/assets/80702e49-adfa-4c7b-a743-ae2a666a208c)

---

### ❗️**O(log n) - Logarithmic Time Complexity**

> **O(log n)은 로그 복잡도(logarithmic complexity)라고 부르며, Big-O표기법중 O(1) 다음으로 빠른 시간 복잡도를 가진다.**
> 

```python
def binary_search(arr, target):
    low = 0
    high = len(arr) - 1
    while low <= high:
        mid = (low + high) // 2
        guess = arr[mid]
        if guess == target:
            return mid
        if guess > target:
            high = mid - 1
        else:
            low = mid + 1
    return None

arr = [1, 3, 5, 7, 9]
target = 5
print(binary_search(arr, target))  # Output: 2 (index of target in arr)
```

![OlogN](https://github.com/user-attachments/assets/661711f1-e0fe-46f0-917d-1b06fe506f82)

---

### ❗️**O(n^2) - Quadratic Time Complexity**

> **O(n2)은 2차 복잡도(quadratic complexity)라고 부르며, 입력값이 증가함에 따라 시간이 n의 제곱수의 비율로 증가하는 것을 의미한다.**
> 

```python
def O_quadratic_algorithm(n):
    for i in range(n):
        for j in range(n):
            # 무언가..
            pass

def another_O_quadratic_algorithm(n):
    for i in range(n):
        for j in range(n):
            for k in range(n):
                # 무언가..
                pass

```

---

![On^2](https://github.com/user-attachments/assets/4eb4e564-b839-41ce-a974-55a3388059fb)

### ❗️**O(2^n) - Exponential Time Complexity**

> O(2n)은 기하급수적 복잡도(exponential complexity)라고 부르며, Big-O 표기법 중 가장 느린 시간 복잡도를 가진다.
> 

**종이를 42번 접으면 그 두께가 지구에서 달까지의 거리보다 커진다는 이야기를 들어보신 적 있으신가?**

**매번 접힐 때마다 두께가 2배 로 늘어나기 때문이다. 구현한 알고리즘의 시간 복잡도가 O(2^n)이라면 다른 접근 방식을 고민해 보는 것이….**
<img width="461" alt="피보나치" src="https://github.com/user-attachments/assets/12a96578-1305-4ed5-94a0-63adaab42e92">

---

# **이분/이진 탐색이란?**

이분/이진 탐색은 탐색 범위를 반씩 좁혀가며 탐색하는 알고리즘이다.

<aside>
❗ 이분 탐색은 결정 문제의 답이 이분적일 때, 그리고 데이터가 **정렬**되어 있을 때 사용할 수 있다.

</aside>

보통 정렬되지 않은 리스트를 탐색해야 할 때 앞에서부터 순차적으로 확인하는 탐색인 순차 탐색을 쓰지만, 원소를 하나씩 확인해야 하기에 시간 복잡도가 **`O(n)`**이다. 

그러나 이분 탐색은 계속해서 탐색 범위를 반으로 줄여나가기에 **`O(logN)`**으로 순차 탐색보다 빠르다. 

![OlogN2](https://github.com/user-attachments/assets/f0d0020c-b022-4787-842e-010531495e1c) ![y=x](https://github.com/user-attachments/assets/8ca4c932-0944-4810-8e19-3642cd84f393)

## 움짤로 보는 이분탐색 이해

![이분탐색](https://github.com/user-attachments/assets/79903930-3106-4823-96e9-7c789d274645)

출처:&nbsp;https://blog.penjee.com/binary-vs-linear-search-animated-gifs/


## 순서대로 탐색해보자!
![이분탐색1](https://github.com/user-attachments/assets/ea00db10-2d03-4b16-a71b-0cb2a3fbed6a)
![이분탐색2](https://github.com/user-attachments/assets/0f15662d-0aec-4ba7-a02b-bc88eebcd9d3)
![이분탐색3](https://github.com/user-attachments/assets/218dcfe7-adfa-45e1-a2c1-cf4eeb0126d9)
![이분탐색4](https://github.com/user-attachments/assets/824db0cb-f76f-4f50-8e0b-95acb926eb00)
![이분탐색5](https://github.com/user-attachments/assets/c854d7ed-fb67-4de4-a8b7-4313d5ddca8a)

## 예시

이분 탐색을 구현하는 방법에는 2가지가 있다.

1. 재귀 함수
2. 반복문

### 재귀 함수

```python
def binary_search(array, target, start, end):
	if start > end:
		return None
	mid = (start + end) // 2
	if array[mid] == target:
		return mid
	elif array[mid] > target:
		return binary_search(array, target, start, mid - 1)
	else:
		return binary_search(array, target, mid + 1, end)
```

### **반복문**

```python
def binary_search(array, target, start, end):
    while start <= end:
        mid = (start + end) // 2
        if array[mid] == target:
            return mid
        elif array[mid] > target:
            end = mid - 1
        else: # array[mid] < target
            start = mid + 1
    return None
```

## 자 이제 풀어볼까?

https://www.acmicpc.net/problem/1920 - 수 찾기

https://www.acmicpc.net/problem/10816 - 숫자 카드 2

https://www.acmicpc.net/problem/2805 - 나무 자르기

---

# 🤑 그리디 알고리즘

**그리디(Greedy) 알고리즘**은 **탐욕법**이라고도 하며, **현재 상황에서 지금 당장 좋은 것만 고르는 방법**을 의미합니다.

- 일반적인 그리디 알고리즘은 문제를 풀기 위한 최소한의 아이디어를 떠올릴 수 있는 능력을 요구합니다.

**그리디 알고리즘**을 이용하면 **매 순간 가장 좋아 보이는 것만 선택하며, 현재의 선택이 나중에 미칠 영향에 대해서는 고려하지 않습니다.**

### 예시

루트 노드에서 출발하여 각 단계에서 가장 큰 값을 선택하는 경로를 따라가는 문제에서, 그리디 알고리즘은 순간순간 최적의 값을 선택합니다.


![그리디 1](https://github.com/user-attachments/assets/708bf42d-92da-4d55-a207-06edcd1c8c5a)

아래처럼 `5` -> `7` -> `9` 로 거쳐가면 `21`이란 최댓값이 나옵니다.

하지만 그리디 알고리즘은 어떻게 갈까요? 놀랍게도 매순간 선택지 중 가장 최적의 해만 고릅니다.

![그리디 2](https://github.com/user-attachments/assets/dcaf4022-3f30-4f64-87b4-d9f798aaabd8)

루트 노드 `5`에서 시작하여 `7`, `10`, `8` 중 가장 큰 `10`을 선택하고, `4`, `3` 중에 `4`를 선택합니다.

<aside>
🔥

일반적인 상황에서 그리디 알고리즘은 최적의 해를 보장할 수 없을 때가 많습니다.

</aside>

하지만 코딩 테스트에서의 대부분의 그리디 문제는 **탐욕법으로 얻은 해가 최적의 해가 되는 상황에서, 이를 추론**할 수 있어야 풀리도록 출제가 된다고 합니다.

그리디 알고리즘은 기준에 따라 좋은 것을 선택하는 알고리즘이므로 문제에서 '가장 큰 순서대로', '가장 작은 순서대로'와 같은 기준을 제시해줍니다.

### 예시 문제

- [ATM](https://www.acmicpc.net/problem/11399)
- [동전 0](https://www.acmicpc.net/problem/11047)

---

### 풀이

- ~~풀기전에 눌러보면 후회함~~
    
    ### ATM
    
    현재 i번째 사람이 인출 할 때 i앞까지 모든 사람의 시간을 총합에 더해주어 최소로 나오는 총합을 구하면 된다.
    
    - 정렬 후 누적합을 통해 계산
    
    ```python
    
    n = int(input())
    sum = 0 
    p = sorted(list(map(int,input().split())))
    
    # 로직
    for i in range(n):
        sum += p[i] * (n-i)
        
    # 출력
    print(sum)
    ```
    
    ### 동전 0
    
    가장 큰 거스름돈 부터 몫과 나머지를 구하면서 최소한의 동전 개수를 구한다.
    
    - 가장 큰 거스름돈 단위부터 나눠야 동전 개수가 제일 적기 때문
    
    ```python
    import sys # 입력을 빨리 받기 위함
    input = sys.stdin.readline
    from collections import deque # deque 자료구조
    
    N, K = map(int, input().split()) # 동전 종류, 가치의 합
    q = deque()
    for i in range(N):
        q.appendleft(int(input())) # 왼쪽에 값 추가
    
    n = 0
    
    while True:
        n = n + (K // q[0])
        K = K % q[0]
        q.popleft()
        if len(q) == 0:
            break
    print(n)
    ```
    

---

# 📌 동적 프로그래밍이란?

<aside>
❓

동적 프로그래밍(Dynamic Programming)은 어떤 문제를 해결하는 방법 중 하나로, 큰 문제를 작은 하위 문제로 나누어 해결하는 알고리즘 설계 기법

</aside>

**이러한 하위 문제들은 한 번만 계산하고, 그 결과를 메모리에 저장해두었다가 필요할 때 재사용하여 중복 계산을 피할 수 있습니다.** 이를 통해 시간 복잡도를 줄이고 효율적으로 문제를 해결할 수 있습니다.

그렇다면 어떤 문제에 DP를 사용하면 될까요?🧐

두가지의 조건을 만족하는지 확인!

- 큰 문제를 작은 문제로 나눌수 있는지?
- 동일한 작은 문제를 반복적으로 작동시켜야 하는지?

DP 알고리즘의 대표 문제는 **피보나치 수열** 문제가 있습니다.

<img width="461" alt="피보나치" src="https://github.com/user-attachments/assets/1b46d75b-748b-4504-b356-8076470e32b3">

피보나치 수를 각각 구해보면 아래와 같습니다:

| n | 피보나치 수열 |
| --- | --- |
| 0 | 0 |
| 1 | 1 |
| 2 | 1 |
| 3 | 2 |
| 4 | 3 |
| 5 | 5 |
| 6 | 8 |
| 7 | 13 |

여기서 하나의 규칙을 발견할 수 있습니다. **앞에 있는 두 수를 더하면** 피보나치를 수를 구할수 있습니다.이러한 **규칙 또는 관계식을 저희는 "점화식"**이라고 부르게 됩니다. 피보나치의 점화식을 구하면 다음과 같습니다.

- *A*1=1 (시작 항)
- *A*2=1 (시작 항)
- *An*=*A*(*n*−1)+*A*(*n*−2)

피보나치 수열의 규칙을 다르게 나타내면 다음과 같습니다.

![image (2)](https://github.com/user-attachments/assets/1d65cc5b-4ee9-4a9b-805f-bde04b9e6531)
 
## 📌 동적 프로그래밍 종류

피보나치 수열을 DP로 푸는 방법은 두가지가 있습니다. 동적 프로그래밍은 크게 "탑다운 방식(메모이제이션)"과 "바텀업 방식(테이블 사용)"으로 구현할 수 있습니다.

### Top Down 방식(메모이제이션)

- **재귀적으로 큰 문제를 작은 문제들로 나누어 해결하면서, 중간 결과들을 메모리에 저장하여 중복 계산을 피합니다.**
- 메모이제이션(memoization)은 계산한 값을 저장하는 캐시를 의미합니다.
- 이미 계산한 하위 문제의 결과를 저장해두고, 같은 하위 문제를 다시 만났을 때 이전에 계산한 결과를 사용합니다.
- 재귀적인 호출로 구현되므로, 스택 오버플로우가 발생할 수 있습니다.

```python
N = int(input())
seq = [0, 1, 1] + [0] * (N - 2)

def fibonacci(x):
    if seq[x]:
        return seq[x]
    seq[x] = fibonacci(x - 1) + fibonacci(x - 2)
    return seq[x]
print(fibonacci(x))
```

### Bottom Up 방식(테이블 사용)

- 작은 문제부터 차례대로 해결하면서 결과를 테이블에 저장합니다.
- 작은 문제의 결과를 이용해 큰 문제를 점진적으로 해결합니다.
- 반복문을 사용하여 구현되므로 스택 오버플로우의 위험은 없습니다.
- 바텀업 방식은 일반적으로 더 선호되는 방식입니다.

```python
N = int(input())
seq = [0, 1, 1] + [0] * (N - 2)

def fibonacci(x):
    for i in range(3, x+1):
        seq[i] = seq[i-1] + seq[i-2]
    return seq[x]
print(fibonacci(x))
```

---

## 예시문제

### 🔗 바로가기 링크

[설탕 배달](https://www.acmicpc.net/problem/2839)

[1로 만들기](https://www.acmicpc.net/problem/1463)

### 풀이

- ~~풀기전에 보면 후회할거임~~
    
    ### 설탕 배달
    
    ```python
    import sys # 입력에 시간 절약을 위함
    input = sys.stdin.readline
    
    N = int(input())
    sum = 0
    while True:
        if N < 0:
            print(-1)
            break
        elif N % 5 != 0: # 5의 배수가 아니면
            N = N - 3
            sum += 1
        elif N % 5 == 0: # 5의 배수면
            sum += N // 5
            print(sum)
            break
    ```
    
    ---
    
    ### 1로 만들기
    
    발상의 전환으로 N에서부터 1로 가는 방법을 찾는 것이아닌 1에서부터 N으로 가는 과정들을 dp테이블에 모두 저장해놓는다.
    
    ```python
    import sys # 입력에 시간 절약을 위함
    input = sys.stdin.readline
    
    N=int(input())
    
    dp=[0,0,1,1] # 초기 테이블
    
    for i in range(4,N+1):
        dp.append(dp[i-1]+1)
    
    		# 경우의 수중에 가장 작은 값을 dp테이블에 저장한다.
        if i%2==0:
            dp[i]=min(dp[i//2]+1,dp[i])
        if i%3==0:
            dp[i]=min(dp[i//3]+1,dp[i])
    
    print(dp[N])
    ```
