# JPA orphanRemoval의 치명적 함정: DELETE 명령이 실종되는 이유

## 들어가며: 편리함이라는 달콤한 함정

JPA `orphanRemoval = true` 옵션을 처음 봤을 때 정말 신기했다. 부모 엔티티의 컬렉션에서 객체를 제거하는 단순한 자바 코드만으로 DB에 DELETE 쿼리가 실행된다니, 이보다 편리한게 어디있나 싶었다. JPA가 알아서 다 해주는 것 같았다.

하지만 이 편리함 뒤에 숨겨진 **Hibernate와의 '복잡하고 엄격한 계약 조건'**을 깨닫기까지 실제 프로젝트에서 며칠 밤을 잠 못 이루며 디버깅해야 했다. 우리가 흔히 사용하는 간결한 로직이 어떻게 DB 삭제 명령을 흔적도 없이 사라지게 만드는지, 그 원인이 무엇인지 알아보자.

---

## 1. orphanRemoval의 작동 원리: "고아"를 감지하는 메커니즘

`orphanRemoval = true`가 설정된 관계에서 JPA는 자식 엔티티를 특별하게 관리한다. 부모 엔티티가 살아있는데 더 이상 자식 객체를 참조하지 않으면 자식을 고아(Orphan)로 간주하고 삭제하는 것이다.

```java
// 부모는 살아있지만, 관계를 끊으면 자식은 고아가 되어 삭제된다.
parent.getChildren().remove(child); // Child는 고아로 간주되어 DELETE!

```

이 기능은 부모 엔티티의 컬렉션 변경이라는 메모리상의 작은 이벤트를 감지하고, 이를 DB의 삭제 명령으로 연결하는 정교한 감시 시스템이 필요하다. 내가 겪었던 버그는 바로 이 감시 시스템의 허점을 건드렸기 때문에 발생했다.

### 1.1. PersistentBag: Hibernate의 특별한 컬렉션

`orphanRemoval`이 어떻게 작동하는지 이해하려면 Hibernate가 컬렉션을 어떻게 관리하는지 알아야 한다.

`parent.getChildren()`로 받는 `List<Child>`는 단순한 `ArrayList`가 아니다. 이건 Hibernate의 **PersistentBag**이라는 프록시 객체다. 이 PersistentBag은 JPA가 부모 엔티티의 생명주기와 컬렉션을 추적하는 데 사용하는 **'태그가 달린 특별한 컨테이너'**라고 볼 수 있다.

PersistentBag의 핵심 특징은 다음과 같다.

1. **변경 감지**: 컬렉션에 요소가 추가되거나 제거될 때 이를 Hibernate에 자동으로 알린다.
2. **소유권 추적**: 이 컬렉션이 어떤 엔티티에 속해있는지 정확히 기록한다.
3. **고아 목록 관리**: `orphanRemoval`이 활성화되면 제거된 요소들을 별도로 추적하여 트랜잭션 종료 시 삭제한다.

### 1.2. CollectionEntry: 감시자의 정체

PersistentBag을 감시하는 게 바로 **CollectionEntry**라는 Hibernate 내부 객체다. CollectionEntry는 영속성 컨텍스트 내부에서 다음 정보를 기록한다.

- 컬렉션의 초기 상태 (스냅샷)
- 현재 상태와의 차이
- 고아가 된 객체 목록 (orphanedElements)
- 컬렉션을 소유한 부모 엔티티 참조

트랜잭션이 커밋되고 `flush()`가 발생하면 Hibernate는 CollectionEntry를 통해 PersistentBag의 변경 사항을 파악하고 필요한 SQL을 생성한다. **이 연결고리가 끊어지는 순간 orphanRemoval은 작동을 멈춘다.**

---

## 2. 문제 상황 1: PersistentBag 인스턴스를 통째로 교체하다

### 2.1. 문제의 코드

프로젝트에서 가장 흔하게 마주치는 패턴이다. Parent의 기존 자식 엔티티들을 전부 삭제하고 새로운 리스트로 교체하는 업데이트 로직이다.

```java
@Service
@Transactional
public class RoutieService {

    public void modifyRoutie(Long parentId, List<RoutiePlace> newPlaces) {
        Parent parent = parentRepository.findById(parentId).orElseThrow();

        // 새로운 자식들로 교체
        Children newChildren = new Children(newPlaces);
        parent.updateChildren(newChildren);  // 여기가 함정!
    }
}

// Parent.java
public class Parent {
    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Child> children = new ArrayList<>();

    public void updateChildren(Children children) {
        this.children = children.getChildren();  // 새로운 List 인스턴스로 교체!
    }
}

```

### 2.2. 증상: 참조 단절 예외

코드상으로는 완벽하게 새 데이터를 넣는 것처럼 보였다. 하지만 트랜잭션이 종료될 때 다음과 같은 예외가 발생했다.

```
org.hibernate.HibernateException:
A collection with orphan deletion was no longer referenced by the owning entity

```

삭제가 문제인데 왜 **'컬렉션 참조'**를 잃어버렸다는 예외가 뜨는 걸까? 이 지점이 내가 며칠 밤을 새운 핵심 고리였다.

### 2.3. 원인: 감시자가 추적하던 대상이 사라졌다

문제의 핵심은 **PersistentBag 인스턴스 자체를 교체**했다는 점이다.

```java
// 트랜잭션 시작 시점
List<Child> children = parent.getChildren();  // PersistentBag 인스턴스 A

// 교체 후
parent.updateChildren(newChildren);  // children 필드가 새로운 ArrayList 인스턴스 B로 교체됨

```

Hibernate의 CollectionEntry는 자신이 감시하던 원본 **PersistentBag 인스턴스 A**의 참조를 순식간에 잃어버린다. 트랜잭션 종료 시 `flush()`가 발생하고 `orphanRemoval` 로직이 실행될 때 Hibernate는 다음과 같이 판단한다.

1. "고아 객체를 찾아 삭제해야 하는데..."
2. "내가 감시하던 PersistentBag 인스턴스가 부모 엔티티에서 사라졌다!"
3. "이건 개발자의 실수다. 예외를 던져야겠다."

이것이 **"A collection with orphan deletion was no longer referenced..."** 예외의 실체다.

### 2.4. 해결 방법: 인스턴스를 유지하고 내용만 조작하라

올바른 방법은 PersistentBag 인스턴스를 절대로 교체하지 않고 그 내부의 내용물만 조작하는 것이다.

```java
// Parent.java (올바른 로직)
public void replaceChildren(List<Child> newChildren) {
    // PersistentBag 인스턴스를 유지하며 기존 요소 제거
    this.children.clear();  // Hibernate에 변경을 알리고 orphanRemoval 감지!

    // 새 요소 추가
    newChildren.forEach(this::addChild);
}

private void addChild(Child child) {
    this.children.add(child);
    child.setParent(this);  // 양방향 동기화
}

```

이렇게 하면 다음과 같은 효과가 있다.

- PersistentBag 인스턴스가 유지된다.
- `clear()` 호출 시 제거된 요소들이 `orphanedElements`에 기록된다.
- 트랜잭션 커밋 시 고아 객체들에 대한 DELETE 쿼리가 정상적으로 실행된다.

---

## 3. 문제 상황 2: Repository 직접 삭제와 영속성 컨텍스트의 충돌

### 3.1. 문제의 코드

두 번째로 흔한 실수는 `orphanRemoval`이 설정된 상태에서 Repository를 통해 직접 삭제하는 것이다.

```java
@Service
@Transactional
public class RoutieService {

    public void modifyRoutie(Long parentId, List<RoutiePlace> newPlaces) {
        Parent parent = parentRepository.findById(parentId).orElseThrow();

        // 1. 기존 자식들의 ID를 수집
        List<Long> oldPlaceIds = parent.getChildren().stream()
                .map(Child::getId)
                .toList();

        // 2. Repository로 직접 삭제 시도 (여기서 함정에 빠졌다)
        childRepository.deleteAllById(oldPlaceIds);

        // 3. 새로운 자식들로 교체
        Children newChildren = new Children(newPlaces);
        parent.getChildren().clear();
        parent.getChildren().addAll(newChildren.getChildren());
    }
}

```

### 3.2. 증상: DELETE 쿼리가 실행되지 않는다

코드상으로는 `deleteAllById()`를 명확히 호출했는데도 DELETE 쿼리가 DB로 전송되지 않거나, 실행되더라도 이후 예기치 않은 동작이 발생한다. 데이터는 DB에 그대로 남아있고 새로운 데이터만 추가되어 중복 레코드가 생성되기도 한다.

### 3.3. 원인: Cascade와 ActionQueue의 작업 취소 메커니즘

문제의 본질은 **Hibernate의 ActionQueue에서 작업이 취소되기 때문**이다. Hibernate의 내부 동작을 살펴보자.

### 3.3.1. CascadeType.PERSIST가 있을 때의 동작

```java
@OneToMany(mappedBy = "parent",
           cascade = {CascadeType.PERSIST, CascadeType.MERGE},
           orphanRemoval = true)
private List<Child> children = new ArrayList<>();

```

위와 같이 `CascadeType.PERSIST`가 설정되어 있으면 부모 엔티티가 영속화될 때 자식 엔티티들도 자동으로 영속 상태로 전이된다.

트랜잭션 실행 흐름은 다음과 같다.

1. **조회 시점**: `findById()`로 Parent를 조회하면 연관된 Child 엔티티들도 영속성 컨텍스트에 로드된다.
2. **삭제 시도**: `childRepository.deleteAllById(oldPlaceIds)` 호출 시 Hibernate는 ActionQueue에 **EntityDeleteAction**을 등록한다.
3. **충돌 감지**: 그런데 문제는 이 Child 엔티티들이 이미 **영속 상태**이고 부모 엔티티의 PersistentBag에 여전히 참조되고 있다는 점이다.
4. **작업 취소**: Hibernate는 다음과 같이 판단한다.
    - "이 Child는 orphanRemoval 대상인데 부모가 여전히 참조하고 있네?"
    - "부모가 명시적으로 관계를 끊지 않았는데 삭제할 수 없다."
    - "EntityDeleteAction을 취소하자."
5. **결과**: DELETE 쿼리가 실행되지 않는다.

### 3.3.2. orphanRemoval의 엄격한 규칙

`orphanRemoval = true`는 단순히 "자식을 삭제해라"가 아니라 **"부모가 관계를 끊었을 때만 자식을 삭제해라"**는 엄격한 규칙이다. Repository를 통한 직접 삭제는 이 규칙을 우회하려는 시도이며 Hibernate는 이를 허용하지 않는다.

영속성 컨텍스트는 다음과 같이 생각한다.

```
삭제 요청 감지 → "잠깐, 이 Child는 orphanRemoval 대상인데?"
규칙 확인 → "부모 컬렉션에서 명시적으로 제거되었나?"
검증 → "parent.getChildren()는 여전히 Child를 참조 중"
결론 → "부모가 버리지 않았는데 함부로 삭제할 수 없다!"
→ 삭제 명령 차단

```

### 3.4. 해결 방법: 컬렉션 조작을 먼저 수행하라

올바른 순서는 다음과 같다.

```java
@Service
@Transactional
public void modifyRoutie(Long parentId, List<RoutiePlace> newPlaces) {
    Parent parent = parentRepository.findById(parentId).orElseThrow();

    // 1. **먼저 PersistentBag에서 제거** (orphanRemoval 감지!)
    parent.getChildren().clear();

    // 2. 새로운 자식들 추가
    newPlaces.forEach(place -> {
        Child child = place.toEntity();
        parent.addChild(child);
    });

    // Repository 직접 삭제는 필요 없음! orphanRemoval이 처리함.
}

```

핵심은 다음과 같다.

1. **PersistentBag의 `clear()` 호출**: 이것이 Hibernate에게 "부모가 관계를 끊었다"는 신호를 보낸다.
2. **orphanRemoval 자동 처리**: `clear()`로 제거된 요소들은 자동으로 고아 목록에 추가되고 트랜잭션 커밋 시 DELETE 쿼리가 실행된다.
3. **Repository 직접 삭제 불필요**: `childRepository.deleteAllById()`를 호출할 필요가 없다. 오히려 충돌을 유발한다.

---

## 4. 도메인 중심 캡슐화: 가장 안전한 패턴

두 가지 문제를 모두 겪은 후 서비스 계층 개발자가 실수로 PersistentBag을 잘못 다루는 것을 원천 차단하는 방법을 고민했다. 그 결과가 바로 **도메인 엔티티 내부에 컬렉션 조작 로직을 캡슐화**하는 것이다.

### 4.1. Parent 엔티티에 안전한 메서드 추가

```java
// Parent.java
@Entity
public class Parent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @OneToMany(mappedBy = "parent",
               cascade = CascadeType.ALL,
               orphanRemoval = true)
    private List<Child> children = new ArrayList<>();

    /**
     * 안전한 자식 교체 메서드
     * PersistentBag 인스턴스를 유지하며 내용만 조작한다.
     */
    public void replaceChildren(List<Child> newChildren) {
        // 1. 기존 자식 제거 (orphanRemoval 감지)
        this.children.clear();

        // 2. 새 자식 추가 및 양방향 동기화
        newChildren.forEach(this::addChild);
    }

    /**
     * 단일 자식 추가 메서드
     * 양방향 관계를 내부에서 자동으로 동기화한다.
     */
    public void addChild(Child child) {
        this.children.add(child);
        child.setParent(this);
    }

    /**
     * 단일 자식 제거 메서드
     */
    public void removeChild(Child child) {
        this.children.remove(child);
        child.setParent(null);
    }
}

```

### 4.2. 서비스 계층은 단순해진다

```java
@Service
@Transactional
public class RoutieService {

    public void modifyRoutie(Long parentId, List<RoutiePlace> newPlaces) {
        Parent parent = parentRepository.findById(parentId).orElseThrow();

        // 엔티티에게 "알아서 안전하게 교체해줘"라고 명령
        List<Child> newChildren = newPlaces.stream()
                .map(RoutiePlace::toEntity)
                .toList();

        parent.replaceChildren(newChildren);

        // 트랜잭션 커밋 시 orphanRemoval이 자동으로 DELETE 실행
    }
}

```

### 4.3. 캡슐화의 장점

이 패턴의 핵심 장점은 다음과 같다.

1. **실수 방지**: 서비스 계층 개발자가 PersistentBag의 존재를 몰라도 안전하게 작동한다.
2. **도메인 중심 설계**: 비즈니스 로직이 엔티티 내부로 이동하여 응집도가 높아진다.
3. **테스트 용이성**: 엔티티의 메서드만 테스트하면 되므로 단위 테스트가 간단해진다.
4. **변경 용이성**: orphanRemoval 관련 로직이 한 곳에 모여있어 수정이 쉽다.

---

## 5. 명시적 관리 패턴: 예측 가능성을 최우선으로

`orphanRemoval`의 두 가지 함정을 모두 경험한 후 다른 접근 방식도 고려해볼 필요가 있다고 느꼈다. 바로 **orphanRemoval을 아예 사용하지 않고 명시적으로 관리하는 패턴**이다.

### 5.1. 명시적 관리 패턴의 구현

```java
// Parent.java - orphanRemoval 제거
@Entity
public class Parent {

    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
    private List<Child> children = new ArrayList<>();

    // 나머지는 동일...
}

// Service.java - 모든 동작을 명시적으로 작성
@Service
@Transactional
public class RoutieService {

    public void modifyRoutie(Long parentId, List<RoutiePlace> newPlaces) {
        Parent parent = parentRepository.findById(parentId).orElseThrow();

        // 1. 명시적 삭제: DB에 직접 쿼리를 날림
        List<Child> oldChildren = new ArrayList<>(parent.getChildren());
        childRepository.deleteAll(oldChildren);

        // 2. 영속성 컨텍스트 동기화
        parent.getChildren().clear();

        // 3. 새로운 자식 추가
        List<Child> newChildren = newPlaces.stream()
                .map(RoutiePlace::toEntity)
                .toList();
        newChildren.forEach(parent::addChild);

        // 4. 명시적 저장 (선택적이지만 더 명확함)
        childRepository.saveAll(newChildren);
    }
}

```

### 5.2. 명시적 관리의 장점

- **예측 가능성**: 모든 동작이 코드에 명시적으로 드러나므로 디버깅이 쉽다.
- **학습 곡선**: PersistentBag이나 CollectionEntry 같은 내부 메커니즘을 몰라도 된다.
- **안정성**: "마법 같은 동작"이 없으므로 예상치 못한 버그가 줄어든다.
- **팀 협업**: 신입 개발자도 코드를 읽고 이해하기 쉽다.

### 5.3. 명시적 관리의 단점

- **코드 길이**: 3~4줄 정도 코드가 더 길어진다.
- **반복**: 유사한 패턴을 여러 곳에서 반복 작성해야 할 수 있다.
- **성능**: 실제로는 거의 동일하지만 이론상 orphanRemoval이 더 최적화될 수 있다.

---

## 6. 결론: 당신의 프로젝트에 맞는 선택을 하세요

두 가지 치명적인 함정을 겪으며 `orphanRemoval`에 대한 관점이 완전히 바뀌었다. 이 기능은 분명 강력하지만 그만큼 신중하게 다뤄야 한다.

### 6.1. 패턴별 비교

세 가지 패턴을 실제 프로젝트 경험에 비추어 비교해보자.

**패턴 1: Repository 직접 삭제 (절대 금지)**

이 패턴은 orphanRemoval과 함께 사용하면 안 된다. DELETE 쿼리가 실행되지 않거나 실행되더라도 ActionQueue에서 취소되어 데이터 불일치를 유발한다. 겉보기에는 간결해 보이지만 실제로는 가장 위험한 패턴이다.

**패턴 2: orphanRemoval + 도메인 캡슐화**

이 패턴은 올바르게 사용하면 매우 우아하다. 코드가 간결하고 도메인 중심 설계를 자연스럽게 유도하며 비즈니스 로직이 엔티티 내부에 응집된다. 하지만 PersistentBag과 CollectionEntry의 동작 원리를 완벽히 이해해야 하며 팀원 모두가 이 규칙을 숙지해야 한다. 한 명이라도 실수하면 디버깅에 많은 시간이 소요된다.

**패턴 3: 명시적 관리**

이 패턴은 가장 보수적이지만 가장 예측 가능하다. 모든 동작이 코드에 명시적으로 드러나므로 3개월 후에 다시 봐도 무슨 일이 일어나는지 즉시 이해할 수 있다. 신입 개발자도 쉽게 따라할 수 있고 버그가 발생해도 원인을 빠르게 파악할 수 있다. 코드가 3~4줄 더 길어지는 것은 팀의 안정성과 비교하면 작은 대가다.

### 6.2. 내가 내린 잠정적 결론

이 글을 쓰면서 한 가지 깨달음을 얻었다. **정답은 없다**는 것이다.

만약 당신의 팀이 JPA에 능숙하고 코드 리뷰가 철저하며 도메인 중심 설계를 추구한다면 `orphanRemoval + 캡슐화` 패턴은 훌륭한 선택이다. 코드는 간결하고 의도는 명확하며 유지보수도 용이하다.

하지만 만약 팀원 중 JPA 초보자가 있거나 프로젝트의 안정성이 코드 간결함보다 중요하다면 `명시적 관리` 패턴을 고려해보는 건 어떨까? 조금 더 길어진 코드가 당신의 팀에게 훨씬 더 큰 평온함을 가져다줄 수 있다.

### 6.3. 마지막 조언

프레임워크의 "마법 같은" 기능은 우리를 유혹한다. 하지만 그 마법이 어떻게 작동하는지 이해하지 못한 채 사용하면 언젠가는 대가를 치르게 된다.

`orphanRemoval`을 사용하기로 결정했다면 반드시 다음을 기억하자.

1. **PersistentBag 인스턴스를 절대 교체하지 말자.** `clear()`와 `addAll()`로 내용만 조작하자.
2. **Repository 직접 삭제를 피하자.** 컬렉션 조작으로 orphanRemoval이 작동하도록 하자.
3. **도메인 엔티티에 로직을 캡슐화하자.** 서비스 계층의 실수를 원천 차단하자.

하지만 만약 이 모든 것이 부담스럽다면 `orphanRemoval`을 과감히 포기하고 명시적 관리 패턴을 사용하는 것도 훌륭한 선택이다. 때로는 단순함이 최선의 전략이다.

