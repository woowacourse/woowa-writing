# 🧠 Android 메모리 관리와 메모리 할당 기법

## 1. Android의 메모리 관리 개요

공식 문서에 따르면 Android 플랫폼은

“사용 가능한 메모리가 있다는 것은 낭비다 (Free memory is wasted memory)”
라는 철학을 기반으로 설계되어 있습니다.

즉, Android 기기는 항상 가능한 모든 메모리를 활용하며,
시스템은 대부분 ‘거의 메모리가 없는 상태’ 에서 동작합니다.

Android Runtime(ART)과 Dalvik VM은 이를 효율적으로 관리하기 위해
페이징(Paging) 과 메모리 매핑(Memory Mapping) 같은 기법을 사용합니다.

이러한 기법을 이해하려면 먼저 프로세스의 메모리 구조와 주소 체계를 살펴볼 필요가 있습니다.

---

## 2. 프로세스 메모리 구조

모든 실행되는 명령어와 데이터는 **CPU가 접근 가능한 메인 메모리(RAM)** 및 **레지스터(Register)** 에 존재해야 합니다.  
또한 각 **프로세스는 독립된 메모리 공간**을 가집니다.

<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/e5cb561b-4329-4587-98df-d6e60675f860" />

| 메모리 영역 | 설명 |
|--------------|------|
| **코드 영역 (Text Segment)** | 프로그램의 실행 명령어(컴파일된 코드)가 저장됨 |
| **데이터 영역 (Data Segment)** | 전역 변수, 싱글턴, 정적 멤버 등 프로그램 시작 시 할당되고 종료 시 해제되는 데이터 |
| **힙 영역 (Heap)** | 실행 중 동적으로 할당되는 메모리 (GC의 대상) |
| **스택 영역 (Stack)** | 함수 호출 시 지역 변수, 매개변수, 반환 주소 등이 저장되며, 함수 종료 시 자동 해제 |

> 💡 Android에서 OutOfMemoryError가 발생하는 경우는 대부분 이 힙 영역이 부족할 때입니다.
---

## 3. 주소 체계

메모리에 저장된 데이터는 **주소(Address)** 로 식별되며, 두 가지 관점이 존재합니다.

| 구분 | 설명 |
|------|------|
| **물리 주소 (Physical Address)** | 실제 RAM 상의 주소 |
| **논리 주소 (Logical Address)** | CPU 및 각 프로세스가 인식하는 주소 |

논리 주소는 실행 중 **물리 주소로 변환**되어야 하며, 이 변환은 **MMU (Memory Management Unit)** 에 의해 수행됩니다.  

Android의 **메모리 매핑(memory mapping)** 은 바로 이 원리를 기반으로 합니다.
<img width="600" height="300" alt="image" src="https://github.com/user-attachments/assets/d75c0330-05ab-435a-a8f4-65b11b36e273" />

---

## 4. 메모리 할당 정책 (Memory Allocation Policy)
<img width="1280" height="287" alt="image" src="https://github.com/user-attachments/assets/ed1ebef6-46cd-41bf-9100-ad78bc411a2c" />

운영체제는 프로세스 실행 시 **필요한 메모리 크기**를 주기억장치에 할당합니다.  
초기에는 **연속 메모리 할당(Contiguous Memory Allocation)** 방식을 사용했습니다.

예:  
> 50MB 크기의 프로세스 → 물리적으로 연속된 50MB 공간 필요

이때 **얼마만큼, 어디에 할당할지** 결정하는 방법이 바로 **메모리 할당 정책**입니다.
초기의 운영체제들은 연속 메모리 할당(Contiguous Memory Allocation) 방식을 사용했습니다.

예: 50MB 크기의 프로세스를 실행하려면 물리적으로 연속된 50MB 공간이 필요함

이때 “얼마만큼, 어디에 할당할지”를 결정하는 방법이 바로 메모리 할당 정책입니다.

### 주요 할당 방식
<img width="600" height="300" alt="image" src="https://github.com/user-attachments/assets/ddb6ab51-3288-479c-9501-38f4ed7abf69" />

| 정책 | 설명 |
|------|------|
| **최초 적합 (First Fit)** | 메모리 공간을 순차적으로 탐색하며 첫 번째로 사용 가능한 공간에 즉시 할당 |
| **최적 적합 (Best Fit)** | 사용 가능한 공간 중 가장 작은 공간에 할당 (전체 탐색 필요) |
| **최악 적합 (Worst Fit)** | 가장 큰 공간에 할당하고, 남은 공간은 다른 프로세스에 분리하여 사용 가능 |

---

## 5. 단편화 (Fragmentation)

### 5.1 외부 단편화 (External Fragmentation)

프로세스의 메모리 할당과 해제가 반복되면, 사용 중인 영역과 빈 영역이 불규칙하게 섞입니다.
이로 인해 총 여유 메모리는 충분하지만,
연속된 큰 공간이 없어 새 프로세스를 적재하지 못하는 현상이 발생합니다.

해결 방법

압축(Compaction)
모든 프로세스를 한쪽으로 몰아 비어 있는 공간을 연속적으로 만드는 방식
단, 성능 비용이 매우 높음

<img width="400" height="250" alt="image" src="https://github.com/user-attachments/assets/909f6651-dacd-4bb3-9483-b07cfef64a53" />

#### 해결 방법
- **압축(Compaction)**: 모든 프로세스를 한쪽으로 몰아 연속 공간을 확보 (단, 비용이 매우 큼)
<img width="600" height="272" alt="image" src="https://github.com/user-attachments/assets/aa5a9164-6e1b-4470-81da-13ea62177812" />

---

### 5.2 내부 단편화 (Internal Fragmentation)
<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/dfcc2755-9b38-4a46-9791-c015264f45e1" />

고정된 단위 크기로 메모리를 할당하는 경우,
프로세스 크기가 단위의 배수가 아닐 때 마지막 블록의 일부가 낭비됩니다.
즉, 실제 사용량보다 약간 더 큰 공간이 할당되어 메모리 일부가 비효율적으로 남는 현상입니다.

---

## 6. 가상 메모리 (Virtual Memory)

<img width="500" height="298" alt="image" src="https://github.com/user-attachments/assets/f91c81d0-0705-4f89-8a77-97979e886ef9" />

**가상 메모리**는 물리 메모리보다 큰 프로세스를 실행하기 위한 기법입니다. 가상 메모리 기법은 프로세스의 **일부만 메모리에 적재** 하고, 나머지는 보조 기억장치에 저장된 상태로 필요할 때 불러옵니다.
이를 통해 실제 물리 메모리보다 큰 프로그램 실행 가능하며 대표적인 구현 방식이 바로 **페이징(Paging)**입니다.

---

## 7. 페이징 (Paging)

페이징은 물리 주소 공간이 **연속적일 필요가 없는** 가상 메모리 관리 기법입니다.  
즉, 연속 메모리 할당 방식의 가장 큰 문제인 **외부 단편화와 압축의 필요성을 해결** 합니다.

### 구현 방식

1. 프로세스의 **논리 주소**를 일정 크기의 **페이지(Page)** 단위로 분할  
2. 메모리의 **물리 주소 공간**을 같은 크기의 **프레임(Frame)** 으로 분할  
3. 각 페이지를 임의의 프레임에 매핑
