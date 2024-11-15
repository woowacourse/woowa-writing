# MultipleBagFetchException 파헤치기

### 개요

반려견 친구 찾기 및 사회화 장려 앱 "반갑개"의 모임(Club) 도메인의 기능을 개발하는 단계에서 마주쳤던 문제에 대해 공유하는 글입니다.
Spring boot 3.3.x / Hibernate 6.x / MYSQL 8.x 이상 버전 기준으로 작성되었습니다.

### 문서 주제
다중 ToMany 연관관계를 포함한 JPA 엔티티 N+1 해결하기

### 대상 독자
Join Fetching의 동작 과정을 이해하고 싶은 개발자.
MultipleBagFetchException을 해결하고자 하는 개발자.

### 배경지식
+ 간단한 SQL 쿼리 및 DML을 이해할 수 있는 개발자.
+ JPA 엔티티, 연관 관계, JPQL 등 기본적인 지식을 알고 있는 개발자.
+ N+1문제를 알고 있는 개발자.
+ Fetch Join, EntityGraph 등을 사용해 본 개발자

  
# 문제 상황


반갑개의 모임(Club) 엔티티는 모임에 참여한 회원(ClubMember), 모임에 참여한 강아지(ClubPet)을 OneToMany 연관 관계를 가지고 있습니다.  

아래는 실제 Club 엔티티를 간략환 예시 엔티티입니다.

```java
@Entity
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@Getter
public class Club {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column
    private String title;

    @OneToMany(mappedBy = "club", cascade = CascadeType.ALL)
    private List<ClubMember> members = new ArrayList<>();

    @OneToMany(mappedBy = "club", cascade = CascadeType.ALL)
    private List<ClubPet> pets = new ArrayList<>();

    public Club(String title) {
        this.title = title;
    }

    public void addMember(Member member) {
        members.add(new ClubMember(this, member));
    }

    public void addPet(Pet pet) {
        pets.add(new ClubPet(this, pet));
    }
}

```

OneToMany의 기본 로딩 전략은 LAZY이며, 반갑개 백엔드 팀의 JPA 연관 관계 컨벤션은 특별한 이유가 없는 한 **지연 로딩(LAZY)** 을 기본으로 사용합니다.  
API 응답에 필요한 연관된 엔티티는 Repository 계층에서 Join Fetching 또는 EntityGraph를 통해 명시적으로 로드하기로 했습니다.  
따라서, MVC 단계 요구사항 중 "내가 참여한 모임 리스트" API를 개발하기 위해 다음과 유사한 JPQL을 사용하게 됐습니다.

  
```java
@Query(value = """
                SELECT C
                FROM Club AS C
                JOIN FETCH C.clubMembers AS CM
                JOIN FETCH C.clubPets AS CP
                WHERE C.id IN (# 내가 참여한 모임 테이블)
List<Club> findAllByParticipatingMemberId(@Param("memberId") Long memberId);
```

위 JPQL 사용하는 서비스의 테스트를 작성하던 중 다음과 같은 예외가 발생했습니다.

![image](tech-write-img/1.PNG)

여러 개의 bag을 Fetch할 수 없다라는 **MultipleBagFetchException**이 발생 했습니다.

### 문제 상황 분석

그렇다면 MultipleBagFetchException은 어떤 예외 일지 알아보겠습니다.  

시작하기 앞서, Spring Data JPA를 통해 ClubRepository를 생성 후 @DataJpaTest로 Club 엔티티에 관한 findAll()을 테스트하며 문제 상황을 재연 했습니다.
```java
@Repository
public interface ClubRepository extends JpaRepository<Club,Long> {

    @EntityGraph(attributePaths = {"members","pets"})
    List<Club> findAll();
}
```
```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE)
class ClubRepositoryTest {

    @Autowired
    private ClubRepository clubRepository;
    @Autowired
    private MemberRepository memberRepository;
    @Autowired
    private PetRepository petRepository;

    @BeforeEach
    void setUp() {
        Member 회원1 = memberRepository.save(new Member("회원1"));
        Member 회원2 = memberRepository.save(new Member("회원2"));
        Member 회원3 = memberRepository.save(new Member("회원3"));
        Member 회원4 = memberRepository.save(new Member("회원4"));
        Member 회원5 = memberRepository.save(new Member("회원5"));

        Pet 강아지1 = petRepository.save(new Pet("강아지1"));
        Pet 강아지2 = petRepository.save(new Pet("강아지2"));
        Pet 강아지3 = petRepository.save(new Pet("강아지3"));
        Pet 강아지4 = petRepository.save(new Pet("강아지4"));
        Pet 강아지5 = petRepository.save(new Pet("강아지5"));

        Club 모임1 = clubRepository.save(new Club("모임1"));
        Club 모임2 = clubRepository.save(new Club("모임2"));

        모임1.addMember(회원1);
        모임1.addMember(회원2);
        모임1.addMember(회원3);
        모임1.addPet(강아지1);
        모임1.addPet(강아지2);
        모임1.addPet(강아지3);

        모임2.addMember(회원4);
        모임2.addMember(회원5);
        모임2.addPet(강아지4);
        모임2.addPet(강아지5);
        
    }

    @DisplayName("Fetch Join")
    @Test
    void findAll() {
        clubRepository.findAll();
    }
}
```
![image](tech-write-img/2.PNG)

문제 상황 재연이 성공적으로 된 것을 확인할 수 있었습니다.  

이제 본격적으로 발생한 **MultipleBagFetchException: cannot simultaneously fetch multiple bags** 에 대해 알아보도록 하겠습니다.  
![image](tech-write-img/3.PNG)

쿼리가 여러 Bag을 **동시에** Fetch 시도하고 있음을 나타내는 예외라고 합니다.  

그렇다면 Bag은 무엇일까요?  
![image](tech-write-img/4.PNG)

결국 Bag이란, 중복 가능한 Set(=MultiSet)을 의미하는 Hibernate의 용어임을 알 수 있었고,  
Java Collection Framework에서는 Bag이 존재하지 않기 때문에 관행상의 이유로 Java Collection Framework의 List로 매핑됨을 알 수 있었습니다.  

**결국 MultipleBagFetchException은 여러 개의 List를 Fetch Join 할 때, 발생되는 문제임을 파악할 수 있게 됐습니다.**

실제로, ClubRepository의 findAll을 다음과 같이 변경하면, 테스트가 성공하게 됩니다.  
![image](tech-write-img/5.PNG)


그렇다면, 왜 Hibernate는 여러 개의 Bag(List)를 fetch Join 하는 것을 명시적으로 금지할까요?

# Bag Fetch Join 동작

```sql
    select
        c1_0.id,
        m1_0.club_id,
        m1_0.id,
        m1_0.member_id,
        c1_0.title 
    from
        club c1_0 
    left join
        club_member m1_0 
            on c1_0.id=m1_0.club_id
```
실제 findAll을 수행했을 때, 생성되는 쿼리는 위와 같습니다. (left join은 EntityGraph로 인한 결과 입니다.)  
해당 쿼리를 실제 MYSQL에서 실행한 결과는 다음과 같습니다. (테스트 코드에 사용된 초기 데이터와 동일합니다.)  

![image](tech-write-img/6.PNG)

query 실행 시 위 처럼 동일한 Club(= 동일한 PK) 가 참여 중인 회원의 인원 수 만큼 중복되어 나오는 것을 확인할 수 있습니다. 
이는 JPA가 Bag을 Fetch Join 하게 될 때, 발생하는 Cartesian product으로 인한 결과입니다.  

# MultipleBagFetchException 발생 원인 

여태까지 나온 정보들을 정리하면 다음과 같습니다.  
  + Bag은 중복을 허용하는 Set이다.(MultiSet)  
  + Hibernate의 Bag은 Java List로 매핑된다.
  + @OneToMany 등 Java List 연관 관계는 실제 데이터베이스에서 Cartesian product이 발생할 수 있다.

즉, 현재 예시에서 참여 중인 회원과 참여 중인 강아지를 Fetch 할 경우, 중복되는 엔티티가 (참여 중인 회원 X 참여 중인 강아지)만큼 나올 수 있게 된다는 의미입니다.  
JPA의 영속성 컨텍스트는 기본적으로  특정 ID를 가진 엔티티가 하나만 존재하도록 보장하는 유일성을 가지고 있습니다. 그리고 영속성 컨텍스트에 저장된 엔티티는 ID를 기준으로 관리되므로, 중복된 ID를 가진 레코드가 반환되면, 첫 번째 레코드만 영속성 컨텍스트에 저장되고, 나머지는 동일한 인스턴스로 간주합니다.  
이 과정에서 여러 개의 Bag을 fetch 할 경우 **순서가 정해지지 않고 무수히 많은 동일 ID를 가진 엔티티를 Hibernate가 올바르게 매핑 할 수 없게 됨을 알 수 있습니다.**  
그 외에도 쿼리 성능 저하, 데이터 중복으로 인한 일관성이 떨어지는 등의 여러 문제가 발생하게 됩니다.  

# 해결방법
### 1. Collection 변경(중복 제거하기)

단편적으로 MultipleBagFetchException을 방지하기 위해서는 중복을 허용하지 않으면 됩니다.  
결국 Hibernate는 List를 *BagType으로 매핑하기 때문에 이를 Set으로 변경하면 문제가 해결 됩니다. 

![image](tech-write-img/7.PNG)

### 2. OrderCoulmn(순서 부여하기)

@OrderColumn은 JPA에서 컬렉션의 요소들을 특정 순서로 저장하고 관리할 수 있게 해주는 어노테이션입니다. 엔티티가 연결된 컬렉션의 순서를 유지할 수 있도록 합니다.  
또, OrderColumn 을 쓰게되면 hibernate 에서는 *ListType 으로 잡게 됩니다. 이로 인해 MultipleBagFetchException을 방지할 수 있습니다. 

![image](tech-write-img/8.PNG)

> *참고 Hibernate Collection Wrapper Class
> | **컬렉션 타입**          | **Hibernate 컬렉션 클래스** | **중복 허용** | **순서 유지** |
> |-------------------------|----------------------------|---------------|---------------|
> | `Collection`, `List`   | `PersistentBag`           | O             | X             |
> | `Set`                  | `PersistentSet`           | X             | X             |
> | `List` + `@OrderColumn`| `PersistentList`          | O             | O             |


### 1,2번의 문제점


1,2 결과를 보면 예외가 발생하지 않고 정상적으로 동작합니다.. 
![image](tech-write-img/9.PNG)

하지만 이 방법은 Cartesian product으로 인한 데이터 중복 문제를 여전히 해결 할 수 없습니다.  
실제 테스트에서 실행된 쿼리를 수행해보면 결과는 다음과 같습니다. 
![image](tech-write-img/10.PNG)

모임1 기준 참여 중인 회원(3) X 참여 중인 강아지(3) 으로 총 9개의 중복 데이터가 발생함을 알 수 있습니다.  
실제 반갑개 기준 Club, Member, Pet에 각 100만건의 데이터를 삽입 후 랜덤으로 Club에 참여하도록 지정했을 때, 위 쿼리는 30초 이상 소요되며,
타임아웃이 발생 해 실제 실행 시간을 측정하기 어려울 정도로 성능 저하가 발생 했습니다.  

### Batch Size 활용

MultipleBagFetchException이 발생하게 된 경위는 연관 엔티티의 N+1 문제를 해결하는 과정에서 발생된 문제입니다.  
@BatchSize 또는 default_batch_fetch_size 설정을 통해 N+1문제를 Where 절의 IN 쿼리를 통해 개선할 수 있습니다.

default_batch_fetch_size를 10으로 설정하여 아래 테스트 코드를 실행해보면 
```java
@DisplayName("N+1 improve by using batchSize")
    @Test
    void findAll() {
        Member 회원1 = memberRepository.save(new Member("회원1"));
        Member 회원2 = memberRepository.save(new Member("회원2"));
        Member 회원3 = memberRepository.save(new Member("회원3"));

        Pet 강아지1 = petRepository.save(new Pet("강아지1"));
        Pet 강아지2 = petRepository.save(new Pet("강아지2"));
        Pet 강아지3 = petRepository.save(new Pet("강아지3"));
        Club 모임1 = clubRepository.save(new Club("모임1"));

        모임1.addMember(회원1);
        모임1.addMember(회원2);
        모임1.addMember(회원3);
        모임1.addPet(강아지1);
        모임1.addPet(강아지2);
        모임1.addPet(강아지3);

        //영속성 컨텍스트 정리
        entityManager.flush();
        entityManager.clear();

        List<Club> clubs = clubRepository.findAll();
        for (Club club : clubs) {
            club.getMembers().forEach(clubMember -> System.out.println(clubMember.getMember().getName()));
            club.getPets().forEach(clubPet -> System.out.println(clubPet.getPet().getName()));
        }
    }
```
![image](tech-write-img/11.PNG)

위 처럼 Club은 심플한 SELECT * FROM club query가 발생하게 되고, 연관 관계에 해당하는 Member, Pet의 경우는 In 쿼리로 개선되어 나가는 것을 확인할 수 있습니다.  
이 방식을 사용하게 되면, 중복된 행 발생을 예방할 수 있고, Fetch Join을 사용하여 N+1을 해결하는 것이 아니기 때문에 MultipleBagFetchException이 발생하지 않게 됩니다.  

# 결론 

결과적으로 MultipleBagFetchException을 해결하기 위해서, 더 나아가 여러 개의 ToMany 연관 관계를 포함하는 엔티티에서 발생하는 N+1을 해결법은 다음과 같이 정리할 수 있을 것 같습니다.  
+ ToOne 연관 관계는 Fetch Join을 한다.
+ List의 크기가 가장 클 가능성이 높은 ToMany 연관 관계를 Fetch Join 한다.
+ 그 외 ToMany 연관 관계를 지닌 List는 batchSize를 통해 성능을 개선한다.

참고로, 글을 쓰는 시점인 반갑개에서는 default_batch_size만을 통해 위 문제를 해결했습니다. 현재는 모임 리스트 API에 관련된 모든 기능에 페이징 처리를 하기 때문입니다.  
(페이징은 Fetch Join과 혼합하여 사용할 경우 OOM 가능성을 내포하고 있습니다.)

JPA를 활용하여 개발하다 보면 Club처럼 ToMany 연관 관계를 다수 지닌 엔티티를 다루게 될 수 있습니다. 
N+1 문제를 해결하기 위해 무작정 Fetch Join을 사용하는 것 보다, Hibernate의 동작 과정을 이해한 후 N+1을 개선할 필요성이 있습니다.  
Club 엔티티를 개발할 당시에는 JPA를 처음 접한 후 얼마 되지 않는 시점이라 "N+1은 Fetch JOIN으로 해결"이라는 공식이 머릿속에 박혀 있었던 것 같습니다.  

# 참고자료

[Hibernate Github](https://github.com/hibernate/hibernate-orm).  
[기억보다 기록을 - 향로님 블로그](https://jojoldu.tistory.com/457).  
[Baeldung-java-hibernate-multiplebagfetchexception](https://www.baeldung.com/java-hibernate-multiplebagfetchexception).  
