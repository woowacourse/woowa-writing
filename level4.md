# 커넥션 풀 고갈 문제 원인 파헤치기

## 목차

1. 문제 상황

2. TCP 연결과 해제 방식

3. HikariCP의 동작 방식

4. 문제 원인

5. 해결 방법

6. socketTimeout 발생 시 HikariCP의 내부 동작

7. 정리

## 1. 문제 상황

JDBC를 사용할 때 socketTimeout 설정은 필수입니다. 하지만 이번 프로젝트 배포 과정에서 이 설정을 실수로 빠뜨렸고, 그 결과 운영 환경에서 예상치 못한 장애를 마주하게 되었습니다.

장애의 시작은 DB 서버가 실행 중인 EC2 인스턴스가 갑자기 다운되면서부터였습니다. 저는 당연히 죽어버린 DB 서버를 재부팅하면 연결이 복구되고 애플리케이션도 다시 정상 작동할 것이라 생각했습니다.

하지만 현실은 달랐습니다. DB 서버가 재부팅되어 정상화되었음에도 불구하고, 애플리케이션은 여전히 멈춰있었습니다. 로그를 확인해보니 다음과 같은 커넥션 풀 고갈 오류가 계속해서 반복되고 있었습니다.

### 1.1. 애플리케이션 로그 분석

```
org.springframework.transaction.CannotCreateTransactionException: Could not open JPA EntityManager for transaction

   ...

Caused by: org.hibernate.exception.JDBCConnectionException: Unable to acquire JDBC Connection

   ...

Caused by: java.sql.SQLTransientConnectionException: HikariPool-1 - Connection is not available, request timed out after 30001ms (total=10, active=10, idle=0, waiting=17)
```

로그의 핵심은 **total=10, active=10, idle=0, waiting=17**입니다. HikariCP의 최대 커넥션 개수를 10개로 설정했는데, 10개의 커넥션이 모두 'Active(사용 중)' 상태로 꽉 차 있고, 반납되지 않고 있다는 뜻입니다. 이 때문에 뒤이어 들어온 17개의 요청은 커넥션을 얻지 못하고 대기하다가 타임아웃(30001ms)으로 실패했습니다.

왜 10개의 커넥션은 반납되지 않는 걸까요? 원인을 파악하기 위해 서버의 스레드 덤프(Thread Dump)를 확인했습니다.

### 1.2. 스레드 덤프(Thread Dump) 분석

스레드 덤프를 분석해보니, 톰캣의 작업 스레드들이 모두 동일한 지점에서 멈춰 있었습니다.

```text
"http-nio-8080-exec-1" #31 [34] daemon ... runnable
   java.lang.Thread.State: RUNNABLE
        at java.net.Socket$SocketInputStream.read(java.base@21.0.8/Socket.java:1099)
        at com.mysql.cj.protocol.ReadAheadInputStream.fill(ReadAheadInputStream.java:91)
        ...
        at com.mysql.cj.jdbc.ClientPreparedStatement.executeQuery(ClientPreparedStatement.java:1058)
        ...
        at org.hibernate.sql.exec.spi.JdbcSelectExecutor.list(JdbcSelectExecutor.java:165)
```

모든 스레드가 `java.net.Socket$SocketInputStream.read` 메서드를 호출하고 멈춰 있습니다. 이는 **DB 서버로부터 응답 패킷이 오기를 기다리고 있는 상태**입니다.

### 1.3. DB 서버 로그 분석

반면, 재부팅된 DB 서버(MySQL)의 로그를 확인해보니 애플리케이션 로그와는 전혀 다른 상황이 기록되어 있었습니다.

```
[System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.42) starting as process 1...
...
[System] [MY-010931] [Server] /usr/sbin/mysqld: ready for connections. Version: '8.0.42'  socket: '/var/run/mysqld/mysqld.sock'  port: 3306
```
로그에서 확인할 수 있듯, DB 서버는 재기동 절차를 마치고 정상적으로 새로운 연결을 받을 준비가 완료된 상태입니다.
여기서 애플리케이션과 DB 서버 간의 상태 불일치가 발생합니다.
DB 서버는 재부팅 과정에서 기존의 TCP 세션 정보를 모두 잃었습니다. 즉, 애플리케이션이 점유하고 있는 커넥션은 DB 입장에서는 존재하지 않는 유효하지 않은 연결입니다.
하지만 애플리케이션은 이 사실을 인지하지 못한 채, 여전히 해당 연결이 유효하다고 판단하여 응답을 대기하고 있습니다. 분명 DB가 종료되었다가 다시 시작되었음에도, 애플리케이션이 연결 단절을 감지하지 못한 이유는 무엇일까요? 이 현상을 이해하기 위해서는 TCP 프로토콜의 연결 해제 과정을 살펴봐야 합니다.

## 2. TCP 연결과 해제 방식

### 2.1. TCP 연결 수립: 3-Way Handshake

클라이언트와 서버 간 TCP 연결은 3-Way Handshake 과정을 통해 이루어집니다. 연결이 되면 이후 데이터를 주고받을 수 있습니다.

### 2.2. TCP 연결 해제: 4-Way Handshake

연결 해제는 4-Way Handshake 과정을 거칩니다. 왜 연결은 3단계인데 해제는 4단계일까요? 서버가 종료 과정에서 아직 전송해야 할 데이터가 버퍼에 남아있을 수 있기 때문입니다. 이를 graceful shutdown이라고 합니다.

클라이언트가 먼저 종료를 요청하는 경우 다음과 같이 진행됩니다.

```
Client                          Server
  |                               |
  |------ FIN (seq=u) ----------->|  [1단계: 클라이언트 종료 요청]
  |                               |
  |<----- ACK (ack=u+1) ----------|  [2단계: 서버 확인]
  |                               |
  |<----- FIN (seq=v) ------------|  [3단계: 서버도 종료 준비 완료]
  |                               |
  |------ ACK (ack=v+1) --------->|  [4단계: 클라이언트 최종 확인]
  |                               |
[TIME_WAIT]                    [CLOSED]
  |
  | (2MSL 대기)
  |
[CLOSED]
```

### 2.3. 정상 종료 vs 비정상 종료

서버가 정상적으로 종료되면 OS 커널이 해당 프로세스가 가진 모든 TCP 소켓에 대해 정리 작업을 수행합니다. 이때 FIN(Finish) 패킷을 보내는데, 이는 이름 그대로 "나 이제 너랑 연결을 끊을거야” 라는 의미의 패킷입니다.

하지만 OS가 비정상 종료되면 어떻게 될까요? 커널이 죽어버렸기 때문에 FIN 패킷을 보낼 수 없습니다. 클라이언트는 서버가 종료되었다는 신호를 듣지 못했으니, 여전히 연결이 살아있다고 착각하게 됩니다. 앞서 언급한 Half-Open Connection이 바로 이 상태입니다.

만약 이 상태에서 클라이언트가 서버에 무언가 요청을 보낸다면 어떻게 될까요? 재부팅된 서버는 과거의 연결 기억이 없으므로 RST(Reset) 패킷을 보냅니다. 이는 "난 너를 모르니 이 연결을 초기화해라"라는 의미의 패킷입니다.

문제는 애플리케이션이 read() 상태로 블로킹되어 있을 때입니다. 클라이언트는 데이터를 보내지 않고 듣기만 하려 하므로, 서버가 RST 패킷을 보낼 기회조차 만들어지지 않습니다. 결국 클라이언트는 응답을 기다리게 됩니다.

### 2.4. TCP Keepalive 메커니즘

OS 레벨에서는 TCP 연결에 대해 Keepalive 요청을 보냅니다. 하지만 리눅스 기본값은 매우 깁니다.

- Keepalive 시작 시간: 7200초 (2시간)
    
- Probe 전송 간격: 75초
    
- Probe 전송 횟수: 9회

그렇다면 OS가 아닌 애플리케이션 레벨, 즉 우리가 사용하는 HikariCP 커넥션 풀은 이러한 연결을 어떻게 관리하고 있을까요?

## 3. HikariCP의 동작 방식

### 3.1. 커넥션 풀의 목적

HikariCP는 이미 TCP 3-Way Handshake를 완료한 연결들을 풀에 보관합니다. 애플리케이션이 DB 작업을 할 때마다 새로운 연결을 맺으면 TCP Handshake, TLS Handshake, DB 인증 등으로 인한 지연이 발생하기 때문입니다.

### 3.2. 커넥션 생성과 대여

HikariCP 설정이 로드되면 `minimumIdle` 개수만큼 커넥션을 생성해 풀에 유지합니다. 애플리케이션이 커넥션 대여를 요청하면 다음과 같이 동작합니다.

1. 풀에 idle 커넥션이 있으면 즉시 반환
    
2. idle 커넥션이 없고 현재 커넥션 개수가 `maximumPoolSize`보다 작으면 새로 생성해서 반환
    
3. 그렇지 않으면 `connectionTimeout`만큼 대기

### 3.3. 헬스체크와 Housekeeping

HikariCP는 `isValid()` 메서드를 통해 헬스체크를 진행합니다. `keepAliveTime`을 설정하면 idle 상태의 커넥션에 대해 주기적으로 헬스체크를 진행합니다. 또한 커넥션 대여 시에도 헬스체크를 수행하여 문제가 있는 커넥션을 걸러냅니다.

하지만 중요한 점은 **이미 빌려준(Active) 커넥션에 대해서는 헬스체크를 하지 않는다는 것**입니다.

## 4. 문제 원인

이제 문제로 돌아가 봅시다. 왜 DB 서버가 재시작되었는데도 커넥션 풀이 계속 응답을 기다렸을까요?

HikariCP에는 `readTimeout`에 해당하는 설정이 없습니다. 커넥션 풀을 점유해서 쿼리를 실행하는 과정에서 지연이 발생해도 HikariCP는 중간에 연결을 끊지 않습니다.

다음과 같은 시나리오가 발생한 것입니다.

1. 애플리케이션이 커넥션 풀에서 커넥션을 가져옴 (Active 상태)
    
2. DB에 쿼리를 전송하고 `socket.read()`를 호출하며 응답 대기
    
3. DB 서버가 비정상 종료 혹은 네트워크 단절 (FIN 패킷 전송 없음)
    
4. 커넥션이 **Half-Open** 상태가 됨
    
5. HikariCP는 Active 상태의 커넥션에 대해 관여하지 않음
    
6. 해당 스레드는 커넥션을 점유한 채로 존재하지 않는 DB 서버의 응답을 무한정 대기
    

애플리케이션 단에서 readTimeout(socketTimeout)을 설정하지 않았다면, 스레드는 영원히 멈춰 있게 되고, 결국 모든 커넥션이 이 상태에 빠져 서비스 장애로 이어집니다.

## 5. 해결 방법

해결책은 간단합니다. HikariCP에서는 `readTimeout`을 지원하지 않으므로 JDBC 드라이버 레벨에서 `socketTimeout`을 설정하면 됩니다.

### 방법 1: JDBC URL에 직접 설정

```yaml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb?socketTimeout=30000
```

### 방법 2: HikariCP를 통해 설정

```yaml
spring:
  datasource:
    hikari:
      connection-timeout: 20000 
      maximum-pool-size: 10
      data-source-properties:
        socketTimeout: 30000      # 쿼리 실행 중 소켓 타임아웃 (30초)
        connectTimeout: 10000 
```

`socketTimeout`을 30초로 설정하면 쿼리 실행 중 30초 동안 응답이 없으면 연결을 끊습니다. 이렇게 하면 DB 서버가 비정상 종료되어도 최대 30초 후에는 문제를 인지하고 예외를 발생시켜 스레드를 해방시킬 수 있습니다.

## 6. socketTimeout 발생 시 HikariCP의 내부 동작

`socketTimeout`을 설정하면 예외가 발생한다는 것은 알겠습니다. 하지만 예외가 발생한다고 해서 문제가 된 커넥션이 풀에서 바로 제거될까요? 혹시 풀로 반환되었다가 나중에 누군가 쓰려고 할 때야 비로소 제거되는 건 아닐까요?

HikariCP의 소스 코드를 통해 이 커넥션이 어떻게 즉시 격리되고 제거되는지 확인해보겠습니다.

### 6.1. 예외 감지와 퇴거(Evict) 마킹

HikariCP가 애플리케이션에 제공하는 커넥션은 실제 JDBC 커넥션이 아니라, 이를 감싼 `ProxyConnection` 객체입니다. 쿼리 실행 중 `SocketTimeoutException`이 발생하면 프록시 객체가 이를 먼저 가로챕니다.

다음은 `ProxyConnection` 클래스의 `checkException` 메서드입니다.

```
// ProxyConnection.java (HikariCP)

final SQLException checkException(SQLException sqle)
{
   var evict = false;
   // ... 

   // 1. 예외가 치명적인지(네트워크 단절 등) 판단합니다.
   if (sqlState != null && sqlState.startsWith("08") || ERROR_STATES.contains(sqlState) || ... ) {
      // 2. 치명적이라면 abort 처리를 위해 플래그를 켭니다.
      evict = true; 
   }

   // ...

   if (evict) {
      // 3. 해당 커넥션 엔트리에 "퇴거(Evict) 대상"이라고 마킹합니다.
      poolEntry.evict("(connection is broken)");
      delegate = ClosedConnection.CLOSED_CONNECTION;
   }

   return sqle;
}

```

`socketTimeout`은 네트워크 레벨의 문제이므로 `isFatal(e)`가 `true`를 반환합니다. 그러면 `abort()` 메서드가 호출되고, 여기서 `poolEntry.evict()`를 실행해 해당 커넥션에 **이 커넥션은 퇴거 대상**이라는 표시를 붙입니다.

### 6.2. 커넥션 반납과 즉시 폐기

애플리케이션에서 예외 처리를 마치고 `connection.close()`를 호출하면, HikariCP는 커넥션을 풀에 반납하는 `recycle` 과정을 수행합니다. 이때 앞서 붙인 퇴거 표시가 핵심적인 역할을 합니다.

다음은 `HikariPool` 클래스의 `recycle` 메서드입니다.

```
// HikariPool.java (HikariCP)

@Override
void recycle(final PoolEntry poolEntry)
{
   metricsTracker.recordConnectionUsage(poolEntry);

   // 1. 아까 예외 발생 시 evict()로 마킹되었는지 확인합니다.
   if (poolEntry.isMarkedEvicted()) {
      // 2. 마킹되었다면 풀로 돌아가지 않고 물리적 연결을 즉시 종료합니다.
      closeConnection(poolEntry, EVICTED_CONNECTION_MESSAGE);
   } else {
      if (isRequestBoundariesEnabled) {
         try {
            poolEntry.connection.endRequest();
         } catch (SQLException e) {
            logger.warn("endRequest Failed for: {},({})", poolEntry.connection, e.getMessage());
         }
      }
      // 3. 문제가 없다면 다시 풀에 반환합니다.
      connectionBag.requite(poolEntry);
   }
}

```

코드를 보면 `poolEntry.isMarkedEvicted()`를 가장 먼저 확인합니다. 아까 `SocketTimeoutException`이 발생했을 때 이미 마킹해두었기 때문에, 이 조건문은 `true`가 됩니다.

결과적으로 `else` 블록을 타고 풀로 돌아가는 것이 아니라, `closeConnection()`이 실행되어 해당 커넥션은 즉시 물리적으로 종료되고 메모리에서 제거됩니다.

### 요약

즉, `socketTimeout` 설정이 있으면 다음과 같이 진행됩니다.

1. JDBC 드라이버가 타임아웃 예외를 던짐
    
2. HikariCP가 이를 감지하고 해당 커넥션에 퇴거 대상임을 표시함
    
3. 트랜잭션 종료 후 반납 시, 풀에 넣지 않고 제거함
    
따라서 별도의 헬스체크를 기다릴 필요 없이, 문제가 생긴 커넥션은 사용되는 즉시 풀에서 제거 됩니다. 이를 통해 시스템은 재부팅 없이도 커넥션을 유지합니다.

## 7. 정리

DB 커넥션 풀 고갈 문제는 TCP의 Half-Open Connection 특성과 HikariCP의 동작 방식(Active 커넥션 관여 안 함), 그리고 JDBC 드라이버의 무한 대기 특성이 결합되어 발생했습니다.

장애 로그와 스레드 덤프가 보여주듯, `socketTimeout` 설정의 부재는 단순한 지연이 아니라 서비스 전체의 불능 상태를 초래할 수 있습니다. JDBC 드라이버 레벨에서 적절한 `socketTimeout`을 설정함으로써, 애플리케이션은 네트워크 장애 상황에서도 스레드를 보호하고 문제가 생긴 커넥션을 스스로 정리할 수 있게 됩니다.

## 참고자료
* [GitHub - HikariCP Wiki: Rapid Recovery](https://github.com/brettwooldridge/HikariCP/wiki/Rapid-Recovery)

