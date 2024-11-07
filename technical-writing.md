제목: JPA를 사용해야 하는 이유

# 태초에 JDBC가 있었으니

---

## 자바의 철학

JDBC의 탄생 배경을 논하기에 앞서, 먼저 자바라는 언어에 대해 알아보려고 합니다. 자바는 “Write Once, Run Anywhere” (WORA) 라는 철학으로 설계된 프로그래밍 언어입니다. 한 번 작성한 코드가 어디에서나 실행될 수 있다는 건 현대의 개발자들에게는 너무나도 당연한 사실이지만, 분명히 그렇지 않던 시절도 있었습니다.

개발자가 작성한 코드는 컴퓨터 위에서 실행됩니다, 그리고 그 컴퓨터가 돌아가기 위해서는 운영체제라는 소프트웨어가 필수적이죠. 컴퓨터는 저마다 다른 운영체제를 가질 수 있고, 그 점이 개발자들에게 아주아주 큰 불편함을 안겨 주었습니다. 각 운영체제는 서로 다른 API를 제공하며, 파일 시스템 접근, 네트워크 통신, 메모리 관리 등의 방식이 다릅니다. 각 플랫폼에 맞게 코드를 작성하고 테스트를 해야 하니, 개발 시간과 비용의 증가는 당연한 결과였습니다.

자바는 개발자들에게 JVM 추상화를 제공하여 이 문제를 해결했습니다. JVM은 자바 프로그램을 실행하기 위한 가상 머신으로, 자바 컴파일러가 생성한 바이트 코드를 운영체제와 하드웨어에 맞게 해석하고 실행하는 역할을 하며, 이를 통해 자바 프로그램이 다양한 플랫폼에서 동일하게 실행될 수 있도록 해줍니다. 이를 통해 개발자는 운영체제와 하드웨어의 세부 사항에 신경 쓰지 않고 개발에 집중할 수 있게 되었습니다. JVM의 등장이 개발자의 작업을 단순화하고, 실행 환경 간의 호환성을 보장하여 개발과 유지보수의 부담을 크게 줄여 준 것이죠.

JVM이 다양한 환경에서 일관성을 제공하듯이, 데이터베이스와의 통신에도 해결책이 필요했습니다. 운영체제가 서로 다른 API를 제공했던 것처럼, 데이터베이스 간에도 불일치가 존재했으며, 이로 인해 애플리케이션이 특정 데이터베이스에 종속되는 문제가 발생했습니다. 그렇게 작성된 코드는 유지보수를 어렵게 하고, 자바의 “Write Once, Run Anywhere” 철학에도 어긋났죠. 이러한 문제를 해결하기 위해 JDBC라는 데이터베이스 접근 표준 API가 도입되었습니다. JDBC는 데이터베이스 간의 차이로 인한 개발의 어려움을 줄이며, 자바의 플랫폼 독립성을 보존하는 역할을 수행했습니다.

## JDBC가 제공하는 편의성

### 드라이버를 통한 데이터베이스 독립성 보장

JDBC 드라이버는 데이터베이스 벤더가 제공하는 구현체로서, JDBC API 호출을 해당 데이터베이스의 네이티브 호출로 변환하는 역할을 합니다. 개발자는 표준 JDBC API를 통해 데이터베이스 작업을 수행하며, 데이터베이스 벤더가 제공하는 드라이버만 적절히 설정하면 됩니다. 이를 통해 같은 자바 코드를 다양한 데이터베이스에서 실행할 수 있으며, 데이터베이스 종류에 종속되지 않고 일관된 방식으로 데이터를 조회하고 조작할 수 있습니다.

다음과 같이 `DriverManager.getConnection()` 를 호출해 데이터베이스 연결을 생성할 수 있습니다.

```java
Connection con = DriverManager.getConnection("jdbc:mysql://localhost:3306/myDb", "user1", "pass");
```

### **SQL 실행의 일관성**

직접 SQL을 실행하는 대신, JDBC에서 제공하는 `Statement` 인터페이스를 통해  `PreparedStatement` , 그리고`CallableStatement` 객체를 사용해 더 안전하고 효율적인 쿼리 실행이 가능해졌습니다.

**Statement**

`Statement` 객체는 동적 SQL 실행을 위해 사용되며, 다음과 같은 세 가지 방식으로 SQL 명령을 실행할 수 있습니다:

- `executeQuery()`  SELECT 쿼리를 실행하여 결과를 조회할 때 사용됩니다.

    ```java
    String selectSql = "SELECT emp_id, name FROM employees";
    ResultSet rs = stmt.executeQuery(selectSql);
    ```

- `executeUpdate()` INSERT, UPDATE, DELETE 쿼리를 실행하여 데이터베이스의 데이터를 수정하거나 추가할 때 사용됩니다.

    ```java
    String updateSql = "UPDATE employees SET name='John Doe' WHERE emp_id=1";
    int rowsAffected = stmt.executeUpdate(updateSql);
    ```

- `execute()` SQL 명령이 ResultSest을 반환할 수도 있고, 업데이트를 수행할 수도 있는 경우에 사용됩니다.

    ```java
    String createTableSql = "CREATE TABLE test_table (id INT PRIMARY KEY, name VARCHAR(100))";
    boolean isResultSet = stmt.execute(createTableSql);
    ```

**PreparedStatement**

처음 등장한 `Statement`는 동적 쿼리를 실행할 수 있었지만, SQL 인젝션 공격에 취약했습니다. `PreparedStatement`는 미리 컴파일된 SQL문을 사용해 파라미터화된 쿼리를 안전하게 실행할 수 있습니다. 이러한 방식은 SQL 인젝션 공격을 예방할 수 있어, 보안성이 높습니다.

```java
String updateSql = "UPDATE employees SET position=? WHERE emp_id=?";
try (PreparedStatement pstmt = con.prepareStatement(updateSql)) {
    pstmt.setString(1, "Lead Developer");
    pstmt.setInt(2, 1);
    int rowsAffected = pstmt.executeUpdate();
    System.out.println("Rows affected: " + rowsAffected);
} catch (SQLException e) {
    e.printStackTrace();
}
```

**CallableStatement**

`CallableStatement`는 데이터베이스에서 저장 프로시저를 호출할 때 사용됩니다. 이 객체는 복잡한 비즈니스 로직을 서버 측에서 처리할 때 주로 활용됩니다.

```java
String callSql = "{call insertEmployee(?, ?)}";
try (CallableStatement cstmt = con.prepareCall(callSql)) {
    cstmt.setString(1, "Ana");
    cstmt.setString(2, "Developer");
    cstmt.execute();
} catch (SQLException e) {
    e.printStackTrace();
}
```

앞서 설명한 세 가지의 `Statement` 인터페이스는 공통적으로 다음과 같은 장점을 제공합니다:

- SQL 코드의 일관성 제공: 동일한 API를 통해 다양한 SQL 작업을 처리할 수 있습니다.
- 자동 리소스 관리: try-with-resources는 사용한 자원을 명시적으로 닫지 않아도 자동으로 닫아주기 때문에, 메모리 누수나 리소스 관리 실수의 위험을 줄일 수 있습니다.
- SQL 오류 처리: SQL 실행 중 발생할 수 있는 오류를 SQLException으로 통일하여 처리할 수 있습니다.
- 성능 최적화: `PreparedStatement` 와 `CallableStatement` 는 SQL 쿼리와 저장 프로시저를 미리 컴파일 하여, 동일한 쿼리를 반복 실행할 때 성능이 향상됩니다.

### **실행 결과 처리**

데이터베이스에서 조회한 결과는 `ResultSet` 객체로 반환되며, 이 객체를 통해 각 컬럼의 데이터를 손쉽게 가져올 수 있습니다. 데이터베이스가 다르더라도 `ResultSet`을 통해 동일한 방식으로 데이터를 처리할 수 있기 때문에, 개발자는 데이터베이스에 상관없이 결과를 일관되게 처리할 수 있습니다.

`ResultSet` 는 SQL 쿼리의 결과를 순차적으로 읽을 수 있는 기능을 제공하며, 다음 코드와 같이 `next()` 메서드를 사용하여 각 행을 탐색합니다. 기본적으로 커서가 처음부터 끝까지 한 방향으로만 이동 가능하기 때문에, 데이터를 대량으로 처리할 경우 메모리 최적화를 고려해야 합니다.

```java
try (ResultSet resultSet = stmt.executeQuery(selectSql)) {
    List<Employee> employees = new ArrayList<>();
    while (resultSet.next()) {
        Employee emp = new Employee();
        emp.setId(resultSet.getInt("emp_id"));
        emp.setName(resultSet.getString("name"));
        employees.add(emp);
    }
}
```

# JDBC를 사랑할 수 없는 100가지 이유

---

## 코드 작성의 번거로움과 오류 가능성

JDBC 코드를 작성할 때 가장 큰 문제점 중 하나는 반복적인 코드 작성과 다양한 예외 처리가 필수적이라는 점입니다. 예를 들어, 데이터베이스 연결 생성, SQL 실행, `ResultSet`에서 데이터 추출, 자원 관리(직접 해제하거나, try-with-resources 구문 활용) 등의 작업을 개발자가 수동으로 처리해야 하므로 번거로움이 발생합니다. 이 작업들에는 다음과 같은 문제점이 있습니다.

### 중복 코드 작성

SQL을 실행하는 과정이 유사할지라도, 매번 SQL 구문을 작성하고 파라미터를 설정하며, 예외 처리를 위한 코드를 반복적으로 작성해야 합니다.

### 오류 가능성

JDBC 코드를 작성하는 과정에서 가장 빈번하게 발생할 수 있는 예외는 `SQLException`입니다. 그러나 이외에도 다양한 예외가 발생할 수 있습니다. 예를 들어, `ResultSet`에서 특정 컬럼이 `null`일 경우, 해당 값을 제대로 처리하지 않으면 `NullPointerException`이 발생할 수 있습니다.

또한, JDBC에서 `Connection`, `Statement`, `ResultSet`과 같은 자원은 명시적으로 해제해야 합니다. 자원 해제를 실수로 누락하면 자원 누수로 인해 데이터베이스 연결이 제대로 해제되지 않고 남아있을 수 있으며, 이는 커넥션 풀 고갈이나 애플리케이션의 성능 저하로 이어질 수 있습니다. try-with-resources 구문을 활용하여 자원 누수를 방지할 수 있지만, 이를 사용하지 않는 경우 자원 해제 누락으로 인한 커넥션 누수 문제가 발생할 수 있습니다. 여전히 SQL 실행 중 발생하는 다양한 예외는 개발자가 직접 처리해야 합니다.

JDBC 코드는 기본적으로 절차 지향적인 방식으로 작성됩니다. 즉, 데이터베이스 연결, SQL 실행, 결과 처리, 자원 해제와 같은 작업을 순차적으로 수동으로 처리해야 합니다. 이 작업들을 올바르게 처리하지 않으면 자원 누수나 실행 오류가 발생할 가능성이 높으며, 이런 반복적이고 복잡한 과정은 JDBC의 주요 단점 중 하나로 꼽힙니다.

## 해결되지 않은 데이터베이스 종속성

JDBC는 데이터베이스 독립성을 제공하기 위한 표준화된 API를 제공하지만, SQL 문법의 차이와 같은 데이터베이스 종속성을 완전히 해결하지는 못했습니다.

`employees`라는 테이블에서 10 건의 레코드만 조회해야 하는 상황이라고 가정해 보겠습니다.

- MySQL에서는 LIMIT 절을 사용하여 조회할 데이터의 개수를 제한합니다.

    ```sql
    SELECT * FROM employees LIMIT 10;
    ```

- Oracle에서는 동일한 결과를 얻기 위해 ROWNUM을 사용해야 합니다.

    ```sql
    SELECT * FROM employees WHERE ROWNUM <= 10;
    ```


JDBC는 다양한 데이터베이스와의 커넥션을 지원하지만, SQL 문법 차이를 해결하기 위해 여전히 데이터베이스에 따라 특정 SQL 문을 작성해야 합니다. 이로 인해 애플리케이션의 데이터베이스 종속성이 발생하며, 데이터베이스 변경 시 애플리케이션 코드의 수정이 필요합니다.

## 데이터베이스와 객체 간의 매핑 문제

JDBC를 통해 데이터베이스에서 조회된 데이터를 애플리케이션에서 활용하려면 객체로 매핑하는 과정이 필요합니다. 그러나 관계형 데이터베이스와 객체지향 프로그래밍의 구조적 차이로 인해 매핑 과정이 복잡해질 수 있습니다.

### 객체와 데이터베이스 간 매핑의 복잡성

예를 들어, 데이터베이스에 다음과 같은 스키마를 가지는 `Employee` 테이블과 `Department` 테이블이 있다고 가정하겠습니다.

```sql
CREATE TABLE departments (
    dept_id INT PRIMARY KEY, -- 부서의 고유 ID
    dept_name VARCHAR(100) -- 부서의 이름
);

CREATE TABLE employees (
    emp_id INT PRIMARY KEY, -- 직원의 고유 ID
    name VARCHAR(100), -- 직원의 이름
    email VARCHAR(100), -- 직원의 이메일 주소
    dept_id INT, -- 직원이 소속된 부서의 ID (외래키)
    FOREIGN KEY (dept_id) REFERENCES departments(dept_id)
);
```

`Employee`와 `Department`는 연관 관계가 있으며, `Employee`는 `Department`에 소속되어 있습니다. 이를 객체로 매핑하기 위해서는 조인 쿼리를 사용하고, 각 테이블의 데이터를 객체로 변환해야 합니다.

```java
String sql = "SELECT e.emp_id, e.name, d.dept_name " +
             "FROM employees e " +
             "JOIN departments d ON e.dept_id = d.dept_id";

try (PreparedStatement pstmt = con.prepareStatement(sql);
     ResultSet rs = pstmt.executeQuery()) {

    List<Employee> employees = new ArrayList<>();
    while (rs.next()) {
        Employee emp = new Employee();
        emp.setId(rs.getInt("emp_id"));
        emp.setName(rs.getString("name"));

        Department dept = new Department();
        dept.setName(rs.getString("dept_name"));
        emp.setDepartment(dept);

        employees.add(emp);
    }
} catch (SQLException e) {
    e.printStackTrace();
}
```

SQL을 통해 조회된 데이터를 객체로 매핑하는 작업이 모두 수동으로 이루어지며, 테이블 간 관계가 복잡해질수록 이러한 매핑 작업 역시 복잡해집니다. 데이터베이스 스키마 변경이나 관계 추가 시, 매핑 로직 또한 함께 수정해야 하기 때문에 유지보수 비용이 높아집니다. 객체는 상속과 다형성을 갖지만, 관계형 데이터베이스는 정규화된 테이블을 사용하므로, 이를 조정하기 위해 수많은 매핑 코드가 필요합니다. 

### 데이터베이스 스키마 변경 시의 코드 수정 부담

JDBC의 또 다른 큰 문제점 중 하나는 데이터베이스 스키마가 변경될 때 발생하는 코드 수정 부담입니다. 데이터베이스 테이블에 새로운 컬럼이 추가되거나 기존 컬럼의 이름이 변경되는 경우, SQL 쿼리뿐만 아니라 해당 데이터를 매핑하는 자바 코드도 함께 수정해야 합니다.

예를 들어, 기존에 `employees` 테이블에서 `name`과 `id` 컬럼만 조회하던 SQL 쿼리에 `email` 이라는 새로운 컬럼이 추가되었다고 가정해 봅시다. 단순히 SQL 쿼리에서만 해당 컬럼을 추가하는 것이 아니라, `Employee` 객체에 `email` 이라는 새로운 필드를 추가하고, 이를 `ResultSet`에서 추출하여 객체로 변환하는 코드도 수정해야 합니다.

```java
String sql = "SELECT e.emp_id, e.name, e.email, d.dept_name " +
             "FROM employees e " +
             "JOIN departments d ON e.dept_id = d.dept_id";

try (PreparedStatement pstmt = con.prepareStatement(sql);
     ResultSet rs = pstmt.executeQuery()) {

    List<Employee> employees = new ArrayList<>();
    while (rs.next()) {
        Employee emp = new Employee();
        emp.setId(rs.getInt("emp_id"));
        emp.setName(rs.getString("name"));
        emp.setEmail(rs.getString("email")); // 이메일 필드 추가
       
        //...
    }
} catch (SQLException e) {
    e.printStackTrace();
}
```

이처럼 데이터베이스의 스키마 변경은 SQL 구문에서뿐만 아니라, 이를 처리하는 모든 로직에서 수정이 필요합니다. 만약 변경 사항이 여러 테이블에 걸쳐 발생한다면, 그에 맞춰 모든 관련 코드들을 수정해야 하며, 이는 버그 발생 가능성을 높이고 유지보수를 어렵게 만듭니다. JDBC는 스키마 변경에 취약하여, 변경 사항을 반영하는 데에 있어 많은 시간이 소요됩니다. 또한, 변경된 스키마에 맞춰 지속적인 수정 작업이 요구되기 때문에, 결국 생산성이 떨어지게 됩니다.

# JPA 한입 해 보세요

---

> JDBC에서의 객체와 관계형 데이터베이스의 차이를 해결하기 위해 ORM(Object-Relational Mapping) 기술이 도입되었습니다. ORM은 데이터베이스와 애플리케이션 객체 간의 매핑을 자동으로 처리하여, 개발자가 직접 SQL을 작성하고 데이터를 매핑하는 수고를 덜어줍니다.
>

## ORM(Object-Realtional Mapping)이란?

ORM은 객체지향 프로그래밍의 객체와 관계형 데이터베이스의 테이블 간의 변환을 자동으로 처리해주는 기술입니다. ORM을 사용하면 객체와 데이터베이스 간의 데이터 매핑을 자동으로 처리할 수 있어, 개발자는 데이터베이스와의 상호작용을 더 직관적이고 간단하게 관리할 수 있습니다. ORM을 사용하면 데이터베이스와의 상호작용을 위한 반복적인 코드 작성을 줄일 수 있으며, SQL 쿼리 작성 대신 객체지향적으로 데이터를 조작할 수 있어, 코드 가독성이 향상된다는 장점이 있습니다.

## JPA의 정의와 역할

JPA는 자바 ORM 기술에 대한 API 표준 명세로, 애플리케이션과 JDBC API 사이에서 동작하는 인터페이스의 집합입니다. JPA를 사용하면 데이터베이스와 상호 작용할 때, 개발자가 직접 SQL을 작성하는 것이 아니라 JPA가 제공하는 API를 호출하여 자동으로 SQL을 생성해 줍니다. 그러면 JPA가 적절한 SQL을 생성하여 데이터베이스에 전달해 주는데, 그 대표적인 예시로 CRUD API가 있습니다.

- **저장 기능(CREATE)**

  `jpa.persist(member);`

  persiste() 메소드를 호출하면, JPA가 객체와 매핑 정보를 보고 적절한 INSERT 쿼리문을 생성하여 데이터베이스에 전달해 객체를 저장합니다.

- **조회 기능(READ)**

  `Member member = jpa.find(Member.class, “memberId”);`

  find() 메소드가 호출되면, JPA는 객체와 매핑 정보를 보고 적절한 SELECT 쿼리문을 생성하여 데이터베이스에 전달하고, Member 객체를 생성하여 반환합니다.

  `Team team = member.getTeam();`

  또한 연관 관계가 있는 경우, 연관된 객체를 사용하는 시점에 적절한 SQL을 실행하여, JDBC 사용 시의 불편함을 해결해 줍니다.

- **수정 기능(Update)**

  `member.setName("변경된 이름");`

  JPA는 별도의 수정 메소드를 제공하지 않는 대신, 조회된 객체의 값을 변경하면 트랜잭션 커밋 시 데이터베이스에 적절한 UPDATE 쿼리를 전달합니다.

- **삭제 기능(Delete)**

  `jpa.remove(member);`

  remove() 메소드를 호출하면, JPA는 매핑된 정보를 바탕으로 적절한 DELETE 쿼리를 생성하여 데이터베이스에 전달합니다.

# JPA 주요 기능

---

CRUD API만으로도 JDBC의 문제인 코드 작성의 번거로움과 오류 가능성을 크게 줄일 수 있지만, 실제로 JPA에서는 그보다 더 많은 기능을 제공합니다.

## 엔티티 매핑

JPA의 사용으로 가장 큰 이점을 얻을 수 있는 부분이 바로 엔티티와 테이블 간의 매핑입니다.

JPA에서는 엔티티 매핑을 위해 다음과 같은 어노테이션들을 제공합니다:
> `@Entity` `@Table` - 객체와 테이블을 매핑합니다.
>
> `@Id` - 테이블의 기본 키를 매핑합니다.
>
> `@Column` - 필드와 데이터베이스의 컬럼을 매핑합니다.
>
> `@ManyToOne` `@JoinColumn` - 연관관계를 매핑합니다.

이전 예시에서 사용된 `employees` 테이블을 다음과 같이 엔티티로 매핑할 수 있습니다.

```java
@Entity
@Table(name = "employees")
public class Employee {
    
    @Id
    @Column(name = "emp_id")
    private Integer id;
    
    @Column(name = "name")
    private String name;
    
    @ManyToOne
    @JoinColumn(name = "dept_id")
    private Department department;
}
```

## 영속성 컨텍스트

영속성 컨텍스트(Persistence Context)는 JPA에서 엔티티 객체의 상태를 관리하는 메모리 저장소입니다. 엔티티 매니저로 저장하거나 조회된 엔티티는 영속성 컨텍스트에 보관되어 관리의 대상이 됩니다. 영속성 컨텍스의 동작에 대해 이해하기 위해서는 먼저 엔티티의 생명주기에 대해 알아야 합니다.

### 엔티티 생명주기

엔티티에는 다음과 같은 네 가지 상태가 존재합니다:

- 비영속(transient): 영속성 컨텍스트와 전혀 관계가 없는 상태입니다.
- 영속(managed): 영속성 컨텍스트에 관리되고 있으며, 데이터베이스와 동기화됩니다.
- 준영속(detached): 영속성 컨텍스트에서 분리되었으며, 더 이상 데이터베이스와 자동으로 동기화되지 않습니다.
- 삭제(removed): 삭제된 상태로, 영속성 컨텍스트에서 제거되었으며, 데이터베이스에서도 삭제됩니다.

![https://media.geeksforgeeks.org/wp-content/uploads/20210626212614/GFGHibernateLifecycle.png](https://media.geeksforgeeks.org/wp-content/uploads/20210626212614/GFGHibernateLifecycle.png)

### 1차 캐시

영속성 컨텍스트는 데이터베이스 기본 키와 매핑된 식별자 값으로 엔티티를 구분하기 때문에, 영속 상태 엔티티는 식별자 값이 반드시 존재해야 합니다. 영속성 컨텍스트는 내부적으로 캐시를 가지고 영속 상태 엔티티들을 관리합니다. 이 캐시는 @Id로 매핑한 식별자를 키, 엔티티 인스턴스를 값으로 가지는 Map의 형식입니다.

```java
Member member = new Member();
member.setId("member1");
member.setUsername("이름");

em.persist(member);
```

위 코드를 실행하면 1차 캐시에 `member1` 을 키값으로 가지는 멤버 엔티티가 저장됩니다.

```java
Member member = em.find(Member.class, "member1");
```

find() 메소드를 호출해서 엔티티를 조회하는 경우, 1차 캐시에 해당 식별자 값이 존재한다면 메모리에 있는 1차 캐시에서 엔티티를 반환합니다. 만약 1차 캐시에 `member1` 을 키값으로 가지는 멤버 엔티티가 존재하지 않는다면, 엔티티 매니저는 데이터베이스를 조회해서 엔티티를 조회하여 생성합니다. 그리고 생성된 엔티티를 1차 캐시에 저장한 후, 영속 상태의 엔티티를 반환합니다.

### **쓰기 지연**

영속성 컨텍스트에는 내부 쿼리 저장소가 존재합니다. 엔티티 매니저는 새로운 엔티티가 등록되는 경우, 데이터베이스에 바로 INSERT 쿼리를 실행하지 않고, 내부 쿼리 저장소에 임시로 저장합니다. 그리고 트랜잭션을 커밋하는 시점에 이 쿼리 저장소에 쌓인 쿼리를 데이터베이스에 전송하며, 이것이 바로 쓰기 지연입니다.

### **변경 감지(Dirty Checking)**

JDBC를 사용하면 수정 쿼리를 직접 작성해야 합니다. 수정 기능은 저장이나 조회, 삭제보다 요구되는 케이스가 더 다양합니다. 엔티티의 경우 이름만 수정할 수도 있고, 이름과 이메일 두 개의 정보를 수정하거나, 별도로 멤버 등급만 수정하는 요구 사항이 추가될 수 있습니다. 세 개의 필드 정보를 모두 업데이트하는 쿼리 하나만을 사용해서 기능을 구현할 수도 있지만, 이럴 경우 실수로 정보를 누락할 가능성이 높습니다. 이를 방지하기 위해 하나의 요구사항마다 그에 맞는 쿼리를 추가하다보면, 수정 쿼리가 점점 많아지고 코드가 복잡해집니다.

JPA에서는 아주 간단하게 엔티티를 수정할 수 있습니다.

```java
Member member = em.find(Member.class, "member1");
member.setUsername("member");
```

별도의 처리 없이 단순히 엔티티를 조회해서 데이터만 변경하면 데이터베이스에 반영됩니다. 이는 JPA가 영속성 컨텍스트 내에 엔티티 인스턴스와 스냅샷을 함께 저장하고 있기에 가능한 것인데요. 엔티티를 영속성 컨텍스트에 보관한 최초의 상태를 스냅샷이라고 합니다. 트랜잭션이 커밋되어 flush()가 호출되는 시점에, 영속성 컨텍스트는 엔티티와 스냅샷을 비교해서 변경된 엔티티에 대한 UPDATE 쿼리가 쓰기 지연 SQL 저장소에 보냅니다. 그리고 쓰기 지연 저장소의 SQL을 데이터에 전송하면, 데이터베이스에도 엔티티 변경 사항이 반영됩니다.

영속성 컨텍스트를 사용함으로써 얻는 이점은 다음과 같습니다:

- 성능 최적화: 영속 상태가 된 엔티티는 데이터베이스를 거치지 않고 1차 캐시에서 바로 불러오기 때문에 성능상 이점을 누릴 수 있습니다.
- 동일성 보장: 식별자가 같을 경우 1차 캐시에서 동일한 인스턴스를 반환하기 때문에 동일성을 보장합니다.
- 쓰기 지연: 데이터베이스에 대한 반복적인 입출력 작업을 줄이고, 여러 엔티티의 변경 사항을 한 번에 모아 처리함으로써 데이터베이스와의 상호작용을 최적화할 수 있니다.
- 변경 감지: 엔티티의 상태가 변경되면, JPA가 이를 자동으로 감지하고 데이터베이스에 적잘한 UPDATE 쿼리를 생성하여 반영합니다. 개발자가 명시적으로 데이터베이스 업데이트 쿼리를 작성할 필요가 없어, 코드 유지보수가 쉬워집니다.

## 객체지향 쿼리 지원

JPQL(Java Persistence Query Language)은 엔티티 객체를 조작하는 데 사용되는 객체지향 쿼리입니다. JPQL은 SQL과 비슷한 문법을 가지면서도, SQL을 추상화하여 특정 데이터베이스에 의존하지 않습니다. 데이터베이스 Dialect만 변경하면 JPQL을 수정하지 않고 손쉽게 데이터베이스를 변경할 수 있다는 장점이 있습니다.

엔티티 직접 조회와 묵시적 조인을 지원해 SQL보다 간결하게 작성이 가능합니다.

```sql
SELECT 
	e.emp_id, 
	e.name, 
	e.email, 
	e.dept_id, 
	d.dept_name
FROM employees e
INNER JOIN departments d ON e.dept_id = d.dept_id
WHERE d.dept_name = 'Engineering'
ORDER BY e.emp_id ASC, e.name ASC;
```

JPQL을 사용하면 위와 같은 SQL을 훨씬 읽기 쉽고 직관적으로 작성할 수 있습니다.

```sql
SELECT e
FROM Employee e
JOIN e.department d
WHERE d.deptName = 'Engineering'
ORDER BY e.empId ASC, e.name ASC;
```

## 스프링 데이터 JPA

스프링 데이터 JPA는 스프링 프레임워크에서 JPA를 편리하게 사용할 수 있도록 지원하는 프로젝트입니다. 이 프로젝트는 리포지토리를 개발할 때 인터페이스만 작성하면, 실행 시점에 구현 객체를 동적으로 생성해서 주입해 주어, 구현 클래스 없이 인터페이스의 작성만으로 애플리케이션이 동작하도록 만들어 줍니다.

### 공통 인터페이스

스프링 데이터 JPA는 간단한 CRUD 기능을 공통으로 처리하는 JpaRepository 인터페이스를 제공하여, 이를 상속하는 인터페이스를 정의하는 것만으로도 기본적인 CRUD 메소드(ex. save, findAll, delete)를 사용할 수 있게 됩니다.

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
}
```

### 쿼리 메소드 기능

스프링 데이터 JPA에서는 메소드 이름만으로 쿼리를 생성하는 기능을 제공하여, 인터페이스에 메소드만 선언하면 적절한 JPQL 쿼리를 생성하여 실행해 줍니다.

```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
		List<Employee> findByNameAndEmail(String name, String email);
}
```

스프링 데이터 JPA가 제공하는 문서 내용에 작성된 네이밍 규칙에 따라서 이름을 작성하면, 대부분의 리포지토리 메소드를 쿼리 작성 없이 실행할 수 있습니다.

# JPA를 사용해야 하는 이유

---

## 객체지향 패러다임 지원

JPA는 데이터베이스 테이블과 자바 객체를 자동으로 매핑하여, 객체를 통해 데이터를 조작할 수 있게 합니다. SQL 대신 객체지향 쿼리 언어인 JPQL을 사용해 객체 중심으로 쿼리를 작성할 수 있습니다. 또한, 객체 간의 관계를 어노테이션으로 명확하게 정의할 수 있어 코드 구조가 직관적입니다. 이러한 방식은 데이터베이스와 애플리케이션 객체 사이의 매핑을 객체지향적으로 처리하고, 비즈니스 로직과 데이터 액세스 로직을 효과적으로 분리하여 코드의 유지보수성을 높입니다.

## 생산성 향상

JPA를 사용하면 SQL 코드를 작성하지 않고도 데이터베이스를 조작할 수 있어, 개발자가 비즈니스 로직 구현에 더욱 집중할 수 있도록 합니다. 스프링 데이터 JPA의 `CrudRepository` 인터페이스를 통해, 기본적인 CRUD 메소드를 제공받아 복잡한 코드 작성 없이 일관된 데이터베이스 작업을 수행할 수 있습니다. 또한, 쿼리 메소드 기능을 이용해 메소드 이름만으로 복잡한 쿼리를 정의할 수 있어, 코드 중복을 줄이고 개발 속도를 더욱 향상시킵니다.

## 성능 개선

JPA는 영속성 컨텍스트를 통해 성능을 크게 향상시킵니다. 1차 캐시는 동일 트랜잭션 내에서 이미 조회된 엔티티를 재사용하여 불필요한 데이터베이스 조회를 최소화합니다. 쓰기 지연 기능은 여러 데이터 변경을 하나의 트랜잭션에서 일괄 처리하여 데이터베이스 쓰기 작업을 보다 최적화합니다. 변경 감지는 엔티티의 상태 변화를 자동으로 감지하여 실제로 변경된 데이터만 업데이트함으로써 불필요한 데이터베이스 작업을 줄입니다. 이 기능들이 모두 함께 작동하여 데이터베이스와의 상호작용을 더욱 효율적으로 만듭니다.

---

### 출처
- https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-1.html
- https://www.baeldung.com/java-jdbc
- https://docs.spring.io/spring-framework/docs/3.0.x/spring-framework-reference/html/jdbc.html
- https://product.kyobobook.co.kr/detail/S000000935744
