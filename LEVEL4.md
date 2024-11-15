# 인덱스 이해하고 적용하기

## 목차

1. [인덱스란 무엇일까?](#인덱스란-무엇일까)
2. [인덱스는 어떻게 사용할까?](#인덱스는-어떻게-사용할까)
3. [인덱스를 어떻게 사용하는 게 좋을까?](#인덱스를-어떻게-사용하는-게-좋을까)

<br>

## 인덱스란 무엇일까?

### 인덱스 개념 및 필요성

#### 개념

추가적인 쓰기 작업과 저장 공간을 활용하여 데이터베이스 테이블의 검색 속도를 향상시키기 위한 자료구조이다.

#### 필요성

- 조건을 만족하는 튜플들을 빠르게 조회하기 위해
- 빠르게 정렬(order by)하거나 그룹핑(group by)하기 위해

인덱스가 걸려있지 않다면 full scan으로 찾아야 한다. 이는 시간복잡도 O(N)이 소요된다.
반면, 인덱스가 걸려있다면 시간복잡도 O(logN)만에 찾을 수 있다. (B-tree 기반 인덱스 기준)

### 인덱스 종류

#### clustered vs. non-clustered

**clustered index**
- 테이블당 단 하나의 클러스티드 인덱스만 존재한다.
    - 따로 지정하지 않으면 PK가 클러스티드 인덱스가 된다.
- DML 실행 시 항상 정렬 상태 유지한다.
    - 물리적으로 정렬된 상태를 유지하기 때문에 검색 속도가 넌클러스티드 인덱스보다 빠르다.
    - 대신 DML 수행 속도가 느리다.

**non-clustered index**
- 정렬이 필요 없다.
- 별도의 장소에 인덱스 페이지를 생성한다.
    - 인덱스 페이지만을 위한 추가 공간이 필요하다.
    - 클러스티드 인덱스보다 데이터 접근 속도가 상대적으로 느리다.

#### B-tree vs. hash

**B-tree index**
- 일반적으로 사용되는 인덱스 알고리즘이다.
- 컬럼의 값을 변형하지 않고, 원래의 값을 이용해 인덱싱한다.

**hash index**
- 주로 메모리 기반의 데이터베이스에서 많이 사용한다.
- 컬럼의 값으로 해시 값을 계산해서 인덱싱한다.

#### 왜 B-tree를 사용할까?

**hash table을 쓰지 않는 이유**

해시 테이블은 데이터에 접근하는 시간복잡도가 O(1)이다. (삽입, 삭제, 조회가 모두 O(1)) 그렇다면 빠르게 조회하기 위해 해시 인덱스를 써야하는 것 아닌가 하는 의문이 들 수 있다.

- rehashing에 대한 부담이 있다.
- 해시 테이블은 등호 연산만 가능하다.(범위 비교 불가능) 따라서, SELECT 질의 조건에 부등호 연산을 써야 한다면 hash table은 문제가 발생한다.
  또한, 해시 테이블은 값을 변형해서 인덱싱하므로, 특정 문자로 시작하는 값으로 검색을 하는 등의 값의 일부만으로 검색하고자 할 때와 정렬에는 사용할 수 없다.
- 복합 인덱스의 경우 전체 attributes에 대한 조회만 가능하다. 일부분만 사용하는 것은 불가능하다.

만약 등호 연산에만 특화된 테이블이 필요하다면 해시를 사용하면 된다.

**B-tree table을 쓰는 이유**

B-tree는 BST를 일반화한 tree이다.

```
이진 탐색 트리 (BST)

- 모든 노드의 왼쪽 서브 트리는 해당 노드의 값보다 작은 값들만 가지고 
  모든 노드의 오른쪽 서브 트리는 해당 노드의 값보다 큰 값들만 가진다.
- 자녀 노드는 최대 두 개까지
```

- B-tree는 자녀 노드의 최대 개수를 늘리기 위해서 부모 노드에 key를 하나 이상 저장한다.
- 모든 leaf 노드들은 같은 레벨에 있다. = balanced tree

<img width="454" alt="image" src="https://github.com/user-attachments/assets/92d314a2-b401-42e7-b074-fad666036845">

**BST vs. B-tree**

- BST
    - 조회, 삽입, 삭제 avg case = O(logN)
    - 조회, 삽입, 삭제 worst case = O(N)
- self-balancing BST
    - 조회, 삽입, 삭제 avg case = O(logN)
    - 조회, 삽입, 삭제 worst case = O(logN)
- B-tree
    - 조회, 삽입, 삭제 avg case = O(logN)
    - 조회, 삽입, 삭제 worst case = O(logN)

**self-balancing BST를 쓰지 않는 이유**

B-tree를 인덱스에 쓰는 이유 중 하나는 B-tree가 balanced tree이기 때문이다. 그렇다면 AVL tree, Red-Black tree와 같은 self-balancing BST도 동일한 시간복잡도를 가지는데 왜 B-tree를 인덱스로 사용할까?

기본적으로 데이터베이스는 secondary storage에 저장된다.

```
secondary storage 특징

- 데이터를 처리하는 속도가 가장 느리다.
- 데이터를 저장하는 용량이 가장 크다.
- block 단위로 데이터를 읽고 쓴다.
```

```
block

- file system이 데이터를 읽고 쓰는 논리적인 단위이다.
	- block의 크기는 2의 승수로 표현되며 대표적인 block size는 4kb, 8kb, 16KB 등이 있다.
- 불필요한 데이터까지 읽어올 가능성이 있다.
```

secondary storage의 특성 때문에 데이터베이스에서 데이터를 조회할 때는 secondary storage에 최대한 적게 접근하는 것이 성능 면에서 좋다.

B-tree는 자녀 노드 개수가 더 많으므로 데이터를 찾을 때 탐색 범위를 빠르게 좁힐 수 있다. 이 때문에 B-tree 인덱스는 self-balancing BST에 비해 secondary storage에 접근을 적게 한다.
또한, 연관된 데이터를 모아서 저장하고 있어 block 단위의 저장 공간을 효율적으로 사용할 수 있다.

## 인덱스는 어떻게 사용할까?

MySQL을 기준으로 사용법을 알아보자.

### 생성

`CREATE INDEX` 구문을 사용하거나, 테이블 생성 시 `CREATE TABLE` 구문에 인덱스를 포함시키는 방법이 있다.

**구문**

```sql
CREATE INDEX 인덱스_이름 ON 테이블_이름(열_이름1, 열_이름2);
```

```sql
CREATE TABLE 테이블_이름 ( 
	열_이름1 열_타입1, 
	열_이름2 열_타입2, 
	INDEX (열_이름1, 열_이름2) 
);
```

<br>

### 삭제

**구문**

```sql
DROP INDEX 인덱스_이름 ON 테이블_이름;
```

> 외래키가 포함된 인덱스를 삭제하려고 하면 에러가 발생한다.
> 
> <img width="1373" alt="image" src="https://github.com/user-attachments/assets/8a9e9727-acb6-4630-8753-042653f4532f">
> 
> 이때는 외래키 제약조건을 먼저 삭제한 후 외래키 삭제를 진행하면 된다.
> 
> **구문**
> 
> ```sql
> ALTER TABLE 테이블_이름 DROP FOREIGN KEY 제약조건_이름;
> ```
> 
> ```sql
> ALTER TABLE 테이블_이름 ADD CONSTRAINT 제약조건_이름 
>     FOREIGN KEY (외래키_열_이름) REFERENCES 참조_테이블_이름 (참조_열_이름);
> ```
> **예시**
> 
> <img width="1302" alt="image" src="https://github.com/user-attachments/assets/21a27440-da33-48e9-87d3-fd8ae6ad5fbb">
> <img width="1279" alt="image" src="https://github.com/user-attachments/assets/5747ef2b-871a-43c9-9be1-4157380b0b2c">
> <img width="1278" alt="image" src="https://github.com/user-attachments/assets/0490b1ae-4e11-4d3c-8745-9191883f60ad">

<br>

### 수정

인덱스를 수정하는 구문은 없다. 인덱스를 수정하려면 기존 인덱스를 삭제한 후, 다시 생성해야 한다.

<br>

### 조회

**구문**

```sql
SHOW INDEX FROM 테이블_이름;
```

**예시**

구문을 통해 member 테이블에 복합 인덱스가 있음을 확인할 수 있다.

<img width="1004" alt="image" src="https://github.com/user-attachments/assets/f7528e5d-0c36-41d7-ae07-bb6b02829c3d">

<br>

### 성능 최적화

쿼리 성능을 향상하고 싶다면 **실행 계획**을 통해 진단해 볼 수 있다. 실행 계획이란 데이터베이스가 데이터를 찾아가는 일련의 과정을 사람이 알아보기 쉽게 DB 결과 셋으로 보여주는 것이다.

**구문**

쿼리 앞에 `EXPLAIN` 키워드를 사용하면 된다.

```sql
EXPLAIN 쿼리;
```

MySQL 8.0.18 부터는 `EXPLAIN ANALYZE`로도 쿼리를 분석할 수 있다. 이는 실행 계획(estimated cost)뿐만 아니라 실제 실행했을 때 비용도 같이 보여준다.

```sql
EXPLAIN ANALYZE 쿼리;
```

> 보통 옵티마이저가 적절하게 인덱스를 선택한다. 하지만 직접 인덱스를 고르고 싶다면 아래 구문을 활용할 수 있다.
> 
> **구문**
> 
> `USE INDEX`는 쿼리에서 특정 인덱스만 사용하도록 힌트를 준다. 여러 인덱스가 있을 때, 특정 인덱스를 선택해서 성능을 개선할 수 있다.
> 
> ```sql
> SELECT .. FROM 테이블_이름 USE INDEX (인덱스_이름) WHERE ..;
> ```
> 
> `FORCE INDEX`는 `USE INDEX`보다 더 강력한 명령어로, 지정된 인덱스를 반드시 사용하게 만든다.
> 
> ```sql
> SELECT .. FROM 테이블_이름 FORCE INDEX (인덱스_이름) WHERE ..;
> ```
> 
> `IGNORE INDEX`는 특정 인덱스를 사용하지 말라고 지시하는 명령어이다. 해당 인덱스를 무시하고 다른 방식으로 데이터를 검색하도록 강제할 수 있다.
> 
> ```sql
> SELECT .. FROM 테이블_이름 IGNORE INDEX (인덱스_이름) WHERE ..;
> ```

</div>
</details>

## 인덱스를 어떻게 사용하는 게 좋을까?
### A. 데이터 양이 많을 때

데이터가 수백 건 이하이면 인덱스를 사용하지 않는 것이 더 효율적일 수 있다.

<br>

**데이터 100건**

- 인덱스 없을 때: 0.001 ms
<img width="1582" alt="image" src="https://github.com/user-attachments/assets/d7628214-b833-425a-a4bb-0d7814570b23">

- 인덱스 있을 때: 0.0022 ms
<img width="1582" alt="image" src="https://github.com/user-attachments/assets/d64ede80-2394-4e0f-b998-229b0ddca982">

**데이터 10만 건**

- 인덱스 없을 때: 71.361 ms
<img width="1582" alt="image" src="https://github.com/user-attachments/assets/002b1911-d2f9-4b75-ad8a-6a5b40873050">

- 인덱스 있을 때: 0.0017 ms
<img width="1582" alt="image" src="https://github.com/user-attachments/assets/b33f6706-adda-49b1-82ab-a339212b16d9">

### B. INSERT, UPDATE, DELETE가 자주 발생하지 않는 컬럼일 때

DML 작업을 할 때 인덱스 관리에 따르는 추가 비용(인덱스 키 추가나 삭제 작업)이 발생한다.

인덱스 키 삭제 작업은 해당 키 값이 저장된 리프 노드를 찾아서 삭제 마크만 하면 돼서 간단하지만, 인덱스 키 추가나 변경 작업은 비용이 많이 든다. (*B-Tree 기준)
- 키 값이 저장될 리프 노드가 꽉 차면 리프 노드를 분리하기 위해 상위 브랜치 노드까지 처리 범위가 넓어진다.
- 키 값에 따라 저장될 리프 노드의 위치가 결정되므로 단순히 키 값만 변경하는 것은 불가능하다. 따라서, 먼저 키 값을 삭제한 후, 다시 새로운 키 값을 추가하게 된다.

참고) 레코드 추가 비용을 1로 가정하고 인덱스 추가 비용을 1.5로 예측하면, 인덱스 추가로 인해 INSERT, UPDATE 문장이 어떤 영향을 받을지 대략적으로 계산할 수 있다.
예. 테이블에 인덱스가 없으면 `1`, 3개면 `5.5(= 1 + 1.5 * 3)` 정도의 비용이 들 것 

### C. WHERE, ORDER BY, GROUP BY 절에 자주 사용되는 컬럼일 때

기본적으로, 자주 조회되는 컬럼에만 인덱스를 생성해야 한다.

복합 인덱스를 생성할 때는 인덱스의 순서에 따라 효율성이 달라질 수 있음을 주의해야 한다.
- 예를 들어, `provider_id`와 `provider_type`으로 복합 인덱스를 생성한다면, `(provider_id, provider_type)` 순이 좋다.
	- `provider_type varchar(255) check (provider_type in ('KAKAO')) not null`
- 예를 들어, `(열_이름1, 열_이름2)`로 복합 인덱스를 생성한다면, 다음의 쿼리는 성능이 개선될 것이다.
    - `SELECT * FROM 테이블_이름 WHERE 열_이름1 = ?;`
    - `SELECT * FROM 테이블_이름 WHERE 열_이름1 = ? AND 열_이름2;`
- 반면, 다음의 쿼리는 성능이 개선되지 않는다.
    - `SELECT * FROM 테이블_이름 WHERE 열_이름2 = ?;`
	- `SELECT * FROM 테이블_이름 WHERE 열_이름1 = ? OR 열_이름2 = ?;`

### D. 카디널리티가 높은 컬럼일 때

예를 들어, 성별 컬럼은 두 가지 경우에 대해서만 데이터가 존재하므로 인덱스를 생성하면 비효율적이다. 값의 범위가 적은 컬럼은 디스크 I/O가 자주 발생하기 때문이다.

### E. Covering Index를 활용한다.

covering index는 조회하는 attributes를 인덱스가 모두 커버한다. 실제 테이블까지 갈 필요가 없기 때문에 조회 성능이 더 빠르다.

<div align="right">

[목차로](#목차)

</div>

## 참고

https://dev.mysql.com/doc/refman/8.4/en/create-index.html

https://youtu.be/bqkcoSm_rCs?feature=shared

https://youtu.be/H_u28u0usjA?feature=shared

https://youtu.be/liPSnc6Wzfk?feature=shared

https://youtu.be/IMDH4iAQ6zM?feature=shared

https://nomadlee.com/mysql-explain-%ec%8b%a4%ed%96%89%ea%b3%84%ed%9a%8d-%ec%82%ac%ec%9a%a9%eb%b2%95-%eb%b0%8f-%eb%b6%84%ec%84%9d/

https://june-coder.tistory.com/64
