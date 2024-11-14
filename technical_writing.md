# 개요

Spring Boot에서는 크게 세 가지 방식으로 채팅을 구현할 수 있다. 비실시간 방식인 폴링, 그리고 실시간 방식인 롱 폴링과 웹소켓이다. 이 글에서는 각각의 방식의 차이를 간단하게 알아보고, 웹소켓을 사용하여 실시간 채팅 프로그램을 구현하는 방법을 설명한다.

### 대상 독자

- 중급 이상의 Java 문법을 이해하고 있는 개발자
- Spring Boot의 기본 작동 원리를 이해하고 있는 개발자

# Spring Boot에서 채팅을 구현하는 방법

## 폴링

폴링(Polling)은 일정한 주기로 HTTP 요청을 보내 데이터를 가져오는 방식이다. 일반적인 HTTP 프로토콜과 동일한 형식으로 요청을 보내고 응답을 받기 때문에, 추가적인 설정 없이 손쉽게 구현할 수 있다. 하지만 요청 주기가 너무 길어지면 채팅 데이터를 받아오는 데 지연이 발생하며, 주기를 너무 짧게 가져가면 서버의 부담이 커진다는 단점이 있다.

## 롱 폴링

롱 폴링(Long polling)은 클라이언트가 요청을 보내면 서버에서는 응답을 주지 않고 기다렸다가, 데이터가 업데이트되었을 때 응답을 보내는 방식으로, 폴링보다 실시간성을 보장하고 싶을 때 사용된다. 하지만 데이터가 자주 바뀌는 경우 폴링보다 더 많은 요청과 응답이 발생하여 부하가 커진다는 단점이 있다. 또한 응답을 보낼 때까지 계속 연결을 유지하기 때문에, 여러 클라이언트가 동시에 요청을 보내면 서버의 부담이 늘어난다.

## 웹소켓

웹소켓(WebSocket)은 서버와 클라이언트 사이의 연결을 계속 유지하여, 실시간 양방향 통신을 가능하게 한다. 또한 헤더의 비중이 큰 HTTP와 달리, 경량화된 데이터를 주고받기 때문에 서버의 부담 또한 줄어든다.

### 연결 과정

![웹소켓 핸드셰이크](image/handshake.png)

1. 클라이언트는 Upgrade 헤더를 포함하여 요청을 보낸다.
2. 서버는 웹소켓 연결이 가능하다는 101 상태 코드를 응답한다(여기까지의 과정을 opening handshake라고 한다.).
3. 클라이언트와 서버는 ws, wss라는 프로토콜로 실시간 양방향 통신을 진행한다.

### ws, wss 프로토콜

ws 프로토콜은 실시간 양방향 통신에서 사용되는 프로토콜이다. 초기 연결 요청을 보낼 때, `http://`가 아닌 `ws://`로 시작하는 주소를 사용해야 한다. https와 같은 secure 연결을 적용하고 싶다면 wss 프로토콜을 사용해서 요청을 보내면 된다.

### 실시간 양방향 통신에서는 어떤 데이터를 주고받는가?

일반적인 HTTP 통신의 요청은 HTTP 메서드, 헤더, 바디 등의 형식이 명확하게 정의되어 있지만, 웹소켓 통신에는 이러한 규격이 없어 임의의 문자열을 주고받을 수 있다. 그러나 STOMP 프로토콜과 같은 기법을 사용하면 일관된 형식으로 데이터를 주고받을 수 있다.

# STOMP 프로토콜

## STOMP란 무엇인가?

STOMP(Simple Text-oriented Messaging Protocol; 문자 기반 메시징 프로토콜)는 웹소켓 통신 규약의 한 종류이다. 웹소켓 자체적으로는 메시지를 어떤 형식으로 주고받을지 정해진 바가 없는데, STOMP를 사용하면 정형화된 형식으로 데이터를 주고받을 수 있다. Spring에서는 STOMP를 공식적으로 지원하기 때문에, 별도의 메시징 프로토콜을 개발하지 않아도 된다.
STOMP(Simple Text-oriented Messaging Protocol; 문자 기반 메시징 프로토콜)는 웹소켓 핸드셰이크 이후의 클라이언트와 서버 사이의 통신 규약을 정의한 것이다. Spring에서는 STOMP를 공식적으로 지원하고 있기 때문에 STOMP를 사용하면 따로 메시징 프로토콜을 개발하지 않아도 되며, 기본으로 제공하는 메시지 큐를 사용하거나 외부 메시지 큐를 사용하는 등 확장성에 용이하다.

## STOMP의 구조

STOMP는 크게 세 가지 요소를 포함한다.

- Command: 메시지의 역할을 나타낸다. `CONNECT`, `SEND`, `SUBSCRIBE` 등 총 15개의 command가 정의되어 있다.
- Header: key-value 형태의 헤더를 지정할 수 있다.
- Body: 메시지의 본문을 담는 부분이다.

```
COMMAND
header1:value1
header2:value2

Body^@
```

## publish - subscribe 구조

STOMP는 publish - subscribe 구조를 지원한다. 따라서 STOMP를 이해하려면 publish와 subscribe가 무엇인지 이해할 필요가 있다.

Publish는 메시지를 전송하는 것이고, subscribe는 메시지를 받기 위해 구독하는 것이다. 메시지는 수신자에게 직접적으로 보내지는 것이 아니고, 일단 메시지 브로커로 메시지를 보내면 메시지 브로커가 구독자들에게 메시지를 대신 전송해주는 방식이다.

# 실시간 채팅 구현 실습

실습 코드는 [Spring 웹소켓 공식 가이드](https://spring.io/guides/gs/messaging-stomp-websocket)를 일부 변형한 것이다.

## 실습 환경

- Java 17
- Spring Boot 3.3.4

## 의존성 추가

`build.gradle`에 다음과 같이 의존성을 추가하면 웹 환경에서 웹소켓을 테스트해볼 수 있다. [Spring Initializr](https://start.spring.io/)를 이용하는 경우 Spring Web과 WebSocket 의존성을 각각 추가하면 된다.

```
implementation 'org.springframework.boot:spring-boot-starter-web'
implementation 'org.springframework.boot:spring-boot-starter-websocket'
```

## 백엔드 소스 코드

### WebSocketConfig.java

```java
@Configuration
@EnableWebSocketMessageBroker
public class WebSocketConfig implements WebSocketMessageBrokerConfigurer {
    @Override
    public void configureMessageBroker(MessageBrokerRegistry config) {
        config.enableSimpleBroker("/topic");
        config.setApplicationDestinationPrefixes("/publish");
    }

    @Override
    public void registerStompEndpoints(StompEndpointRegistry registry) {
        registry.addEndpoint("/ws-connect");
    }
}
```

웹소켓을 사용하기 위해서는 먼저 설정 파일을 작성해야 한다. `configureMessageBroker()` 메서드에서 어떤 메시지 브로커를 사용할지 지정할 수 있다. 예제에서는 스프링이 기본적으로 제공하는 인메모리 메시지 브로커를 사용하고, 구독 URI의 접두어를 `/topic`으로 지정했다. `setApplicationDestinationPrefixes()`는 메시지를 발행할 때 쓰이는 URI의 prefix를 지정한다.

`registerStompEndpoints()` 메서드에서는 웹소켓 초기 연결에 쓰이는 엔드포인트를 지정할 수 있다. 클라이언트는 `wss://example.com/ws-connect` 주소로 `Upgrade` 헤더를 포함하여 요청을 보내 오프닝 핸드셰이크를 진행할 수 있다.

### MessageRequest.java

```java
public record MessageRequest(String username, String content) { }
```

메시지 송신에 사용될 요청 DTO이다.

### MessageResponse.java

```java
public record MessageResponse(String content) { }
```

메시지 수신에 사용될 응답 DTO이다.

### MessageController.java

```java
@RestController
public class MessageController {
    @MessageMapping("/chat")
    @SendTo("/topic/chat")
    public MessageResponse sendMessage(MessageRequest message) {
        if (message.content().equals("error")) {
            throw new RuntimeException("You can't use that word.");
        }

        return new MessageResponse(message.username() + ": " + message.content());
    }
}
```

가장 핵심이 되는 컨트롤러 클래스이다. 일반적인 HTTP 요청을 처리할 때는 `@GetMapping`이나 `@PostMapping` 등의 어노테이션을 사용했을 것이다. 웹소켓에서는 `@MessageMapping`이라는 웹소켓 전용 어노테이션을 사용한다. `@MessageMapping`의 인자값으로는 메시지를 발행할 주소를 지정해야 하는데, 설정 파일의 `setApplicationDestinationPrefixes()` 메서드에서 설정한 값이 메시지 발행 주소의 접두어가 된다. 예제에서는 접두어를 `/publish`로 설정했기 때문에, `/publish/chat` 주소로 메시지가 발행되면 `sendMessage()` 메서드가 호출된다.

`@SendTo` 어노테이션은 메시지의 도착지 주소를 지정하는 역할을 한다. `sendMessage()` 메서드가 호출되면 `/topic/chat` 주소를 구독하고 있는 모든 클라이언트에게 반환값이 전송된다.

## 프론트엔드 소스 코드

JavaScript나 HTML에 익숙하지 않다면 복사, 붙여넣기만 하고, 동작 흐름을 파악하는 참고 자료로만 활용하면 된다.

### app.js

```jsx
const stompClient = new StompJs.Client({
  brokerURL: 'ws://localhost:8080/ws-connect'
});

stompClient.onConnect = (frame) => {
  setConnected(true);
  console.log('Connected: ' + frame);

  stompClient.subscribe('/topic/chat', (greeting) => {
    let message = JSON.parse(greeting.body).content;
    showChat(message);
  });
};

stompClient.onWebSocketError = (error) => {
  console.error('Error with websocket', error);
};

stompClient.onStompError = (frame) => {
  console.error('Broker reported error: ' + frame.headers['message']);
  console.error('Additional details: ' + frame.body);
};

function setConnected(connected) {
  $("#connect").prop("disabled", connected);
  $("#disconnect").prop("disabled", !connected);
  if (connected) {
    $("#conversation").show();
  }
  else {
    $("#conversation").hide();
  }
  $("#greetings").html("");
}

function connect() {
  stompClient.activate();
}

function disconnect() {
  stompClient.deactivate();
  setConnected(false);
  console.log("Disconnected");
}

function sendName() {
  stompClient.publish({
    destination: "/publish/chat",
    body: JSON.stringify({'username': $("#na").val(), 'content': $("#name").val()})
  });
}

function showChat(message) {
  $("#greetings").append("<tr><td>" + message + "</td></tr>");
}

$(function () {
  $("form").on('submit', (e) => e.preventDefault());
  $( "#connect" ).click(() => connect());
  $( "#disconnect" ).click(() => disconnect());
  $( "#send" ).click(() => sendName());
});

```

웹 페이지로 테스트해보기 위한 JavaScript 코드이다. `ws://localhost:8080/ws-connect`로 요청을 보내 초기 연결을 하고 있는 것을 확인할 수 있다. `/topic/chat` 엔드포인트를 구독해두었다가, 응답을 받으면 `showChat()` 함수를 통해 실시간으로 채팅 메시지를 화면에 렌더링한다.

### index.html

```html
<!DOCTYPE html>
<html>
<head>
  <title>Hello WebSocket</title>
  <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/3.3.7/css/bootstrap.min.css" integrity="sha384-BVYiiSIFeK1dGmJRAkycuHAHRg32OmUcww7on3RYdg4Va+PmSTsz/K68vbdEjh4u" crossorigin="anonymous">
  <link href="/main.css" rel="stylesheet">
  <script src="https://code.jquery.com/jquery-3.1.1.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/@stomp/stompjs@7.0.0/bundles/stomp.umd.min.js"></script>
  <script src="/app.js"></script>
</head>
<body>
<noscript><h2 style="color: #ff0000">Seems your browser doesn't support Javascript! Websocket relies on Javascript being
  enabled. Please enable
  Javascript and reload this page!</h2></noscript>
<div id="main-content" class="container">
  <div class="row">
    <div class="col-md-6">
      <form class="form-inline">
        <div class="form-group">
          <label for="connect">WebSocket connection:</label>
          <button id="connect" class="btn btn-default" type="submit">Connect</button>
          <button id="disconnect" class="btn btn-default" type="submit" disabled="disabled">Disconnect
          </button>
        </div>
      </form>
    </div>
    <div class="col-md-6">
      <form class="form-inline">
        <div class="form-group">
          <label for="name">name</label>
          <input type="text" id="na" class="form-control">
          <label for="name">content</label>
          <input type="text" id="name" class="form-control">
        </div>
        <button id="send" class="btn btn-default" type="submit">Send</button>
      </form>
    </div>
  </div>
  <div class="row">
    <div class="col-md-12">
      <table id="conversation" class="table table-striped">
        <thead>
        <tr>
          <th>Greetings</th>
        </tr>
        </thead>
        <tbody id="greetings">
        </tbody>
      </table>
    </div>
  </div>
</div>
</body>
</html>

```

### 웹 테스트

서로 다른 두 개의 탭에서 [http://localhost:8080/](http://localhost:8080/%EC%97%90)에 접속한 뒤, Connect를 누르고 메시지를 전송해 보자. 채팅 메시지가 서로 실시간으로 반영되는 것을 확인할 수 있다.

![image/websocket.png](image/websocket.png)

# 예외 처리 추가하기

### 메시지 발행 과정에서의 예외 처리

웹소켓 통신에서 발생하는 예외는 일반적인 `@ExceptionHandler`로는 처리할 수 없고, 웹소켓 전용 예외 핸들러인 `@MessageExceptionHandler`를 사용해야 한다. 컨트롤러 메서드를 약간 변형해서 채팅 메시지로 금지어를 입력했을 때 예외를 발생시키는 로직을 작성해 보자.

```java
@RestController
public class MessageController {
    private static final Logger log = LoggerFactory.getLogger(MessageController.class);
    private static final Set<String> bannedWords = Set.of("dislike", "hate", "despise");

    @MessageMapping("/chat")
    @SendTo("/topic/chat")
    public MessageResponse sendMessage(MessageRequest message) {
        if (bannedWords.contains(message.content())) {
            throw new RuntimeException("You can't use that word.");
        }
        return new MessageResponse(message.username() + ": " + message.content());
    }

    @MessageExceptionHandler
    public void handleException(RuntimeException e) {
        log.info("Exception: ", e);
    }
}
```

금지어를 전송하면 예외가 발생하며, 다른 사용자에게 메시지가 전달되지 않는 것을 확인할 수 있다.

### 구독 과정에서의 예외 처리

웹소켓으로 전송되는 메시지를 가로채는 인터셉터를 구현하여 구독 과정에서의 예외 처리 로직을 추가할 수 있다.

```java
@Component
public class WebSocketInterceptor implements ChannelInterceptor {
    @Override
    public Message<?> preSend(Message<?> message, MessageChannel channel) {
        StompHeaderAccessor accessor = StompHeaderAccessor.wrap(message);
        if (accessor.getCommand() == StompCommand.SUBSCRIBE) {
            if (authorizationFails()) {
                throw new AuthException("채팅방에 입장할 권한이 없습니다.");
            }
        }
        return message;
    }

    private boolean authorizationFails() {
        // TODO: 인가 관련 로직 구현
        return true;
    }
}
```

```java
// WebSocketConfig.java
@Autowired WebSocketInterceptor webSocketInterceptor;

@Override
public void configureClientInboundChannel(ChannelRegistration registration) {
    registration.interceptors(webSocketInterceptor);
}
```

`ChannelInterceptor`의 구현체를 만들고, STOMP에서 구독에 사용되는 `SUBSCRIBE` 커맨드로 들어오는 메시지를 가로채 특정 조건에 따라 예외를 발생시키도록 작성하였다. 해당 인터셉터를 설정 파일에 등록하면 예외를 처리 로직이 정상 작동한다.

# 정리

지금까지 웹소켓와 STOMP 프로토콜의 개념에 대해 알아보고, 예제를 통해 실시간 채팅 애플리케이션을 만들어 보았다. 만약 기능을 더 추가하거나 성능을 개선하고 싶다면 다음 사항들을 고려해볼 수 있다.

### 여러 개의 채팅방

예제에서는 하나의 채팅방만 사용할 수 있었지만, 발행 주소와 구독 주소에 식별자를 추가하면 여러 개의 채팅방을 구현할 수도 있다.

### 웹소켓이 지원되지 않는 브라우저

일부 오래된 브라우저에서는 웹소켓이 지원되지 않을 수 있는데, 이런 경우 socket.io나 SockJS 도입을 고려해볼 수 있다. socket.io는 웹소켓이 지원되지 않는 브라우저에서 소켓 연결에 실패하면 다른 방식으로 연결할 수 있도록 fallback 기능을 제공한다. SockJS 역시 웹소켓 → HTTP 스트리밍 → 롱 폴링 순서대로 fallback하는 기능을 제공한다.

### 외부 메시지 브로커 도입

예제 코드에서는 인메모리 메시지 브로커를 사용하고 있다. 만약 성능 개선이나 추가적인 기능 사용이 필요하다면 RabbitMQ, Kafka 등 외부 메시지 브로커 도입을 고려해볼 수 있다.

### 데이터베이스 저장

웹소켓이 끊기면 메시지를 주고받을 수 없기 때문에, 웹소켓이 끊겨 있는 동안 전송된 채팅 메시지를 조회하기 위해서는 데이터베이스 저장이 필요하다. 채팅은 insert가 많이 일어나기 때문에 MongoDB 등의 NoSQL을 사용하거나, 일정 기준을 정해서 batch insert하는 방법을 사용하면 좋다.
