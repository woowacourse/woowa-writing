# 1. 무중단 배포란?

## 무중단 배포 개념

무중단 배포는 다운타임 없이 사용자 환경을 방해하지 않고 애플리케이션을 설치하거나 업그레이드하는 것을 목표로 합니다.
다운타임이란 배포하는 동안 사용자가 서비스를 이용하지 못하는 시간을 의미합니다.

## 왜 무중단 배포를 해야 할까?

무중단 배포를 하지 않는다면 새 버전이 실행되기 전에 애플리케이션을 중지해야 하므로 사용자는 적게는 몇 초부터 많게는 몇 분까지 다운타임을 경험하게 됩니다.

또한 **새 버전에서 애플리케이션에 버그가 발생하면 사용자가 쉽게 되돌릴 수 없는 상황**이 생길 수 있으며, 롤백 시 더 많은 다운타임이 발생할 수 있습니다.

# 2. 무중단 배포 전략

## 1. Blue-Green 배포

Blue-Green 배포는 소프트웨어 배포 전략의 하나로, 새로운 버전의 애플리케이션을 안전하게 출시하기 위해 두 개의 환경(Blue와 Green)을 사용하는 방법입니다. 이 방식의 주요 목적은 다운타임을 최소화하고, 롤백을 쉽게 할 수 있도록 하는 것입니다. 아래에 Blue-Green 배포의 상세한 구성 요소와 절차를 설명하겠습니다.

### 1. 환경 구성

- **Blue 환경**: 현재 운영 중인 애플리케이션 버전이 배포된 환경입니다.
- **Green 환경**: 새로 배포할 애플리케이션 버전이 준비된 환경입니다. 이 환경은 Blue와 동일한 설정을 갖추고 있으며, Blue에서의 변경 사항이 Green에 반영됩니다.

### 2. 배포 과정

1. 애플리케이션의 **새로운 버전을 Green 환경에 배포**합니다. 이 과정에서 데이터베이스 마이그레이션도 함께 이루어질 수 있습니다.
2. 새 버전이 Green 환경에서 올바르게 작동하는지 검증하기 위해 테스트를 수행합니다. 이 단계에서 성능 테스트, 통합 테스트 등이 포함될 수 있습니다.
3. 새 버전이 안정적임을 확인한 후, **클라이언트의 요청을 Blue 환경에서 Green 환경으로 전환**합니다. 이 과정은 로드밸런서를 통해 이루어지며, 이때 전체 트래픽을 전환하지 않고 일부 트래픽을 먼저 Green 환경으로 보내어 문제가 없는지 확인할 수도 있습니다.
4. 전환 후, 애플리케이션의 성능과 안정성을 모니터링합니다. 만약 문제가 발생하면 빠르게 Blue 환경으로 롤백할 수 있습니다.

![blue-green](image/blue-green.png)

### 3. 장점

- 새 버전에서 문제가 발생할 경우, Blue 환경으로 즉시 롤백할 수 있습니다.
- 실제 사용자 트래픽을 발생시키기 전에 새 버전을 안전하게 테스트할 수 있습니다.
- 한 번에 트래픽을 모두 새로운 버전으로 옮기기 때문에 **버전 간 호환성을 보장하지 않아도 가능합니다.**

### 4. 단점

- 두 개의 환경을 동시에 운영해야 하므로 인프라 비용이 증가할 수 있습니다.
- 관리해야 할 환경이 두 개이기 때문에 설정과 모니터링이 복잡해질 수 있습니다.

## 2. Rolling 배포

### 1. 개념

**Rolling 업데이트**는 여러 개의 애플리케이션 인스턴스가 있을 때 적용되는 배포 전략입니다. **처음에는 모든 인스턴스가 이전 버전을 실행**하며, 다운타임 없이 점진적으로 새 버전으로 전환합니다.

### 2. 배포 과정

1. **이전 버전을 실행하는 인스턴스 중 하나를 제거**하여, 업데이트된 버전과 이전 버전을 동시에 지원합니다. 이를 위해 **두 버전 간 호환성**이 보장되어야 합니다.
2. 새 인스턴스를 추가하고 이전 버전을 계속 제거하는 방식으로 배포를 진행합니다. 이 과정을 반복하여 모든 인스턴스가 업데이트된 버전으로 교체됩니다.
3. 마지막 인스턴스가 업데이트된 인스턴스로 교체되면 배포가 완료됩니다. 애플리케이션이 예상대로 작동하는지 확인하고, 필요시 테스트를 수행합니다.

![rolling1](image/rolling1.png)

![rolling2](image/rolling2.png)

### 3. 장점

- 배포 중 문제가 발생하면, 모든 변경 사항을 롤백할 수 있어 안전한 배포가 가능합니다.
- 업데이트할 인스턴스 수를 설정할 수 있어, **대규모 배포**에서도 효율적으로 관리할 수 있어 유연합니다.

### 4. 단점

- 배포가 진행되는 동안 구버전과 신버전이 공존하기 때문에 호환성 문제가 발생할 수 있습니다.

## 3. Canary 배포

### 1. 개념

Canary 배포는 점진적으로 업데이트를 배포하여, 처음에는 일부 사용자에게만 새로운 버전을 적용하는 방식입니다. 
점진적으로 구버전에 대한 트래픽을 신버전으로 옮기는 것은 Rolling 배포 방식과 비슷하지만 Canary 배포의 주된 목적은 위험을 줄이고 시스템의 안정성을 높이는 것입니다.

### 2. 배포 과정

1. 처음에는 전체 사용자 중 일부(예: 25%)에게만 업데이트를 배포합니다.
2. 초기 업데이트가 완료되면, 시스템을 테스트하여 애플리케이션이 정상적으로 작동하는지 확인합니다. 이 단계에서 발생하는 문제는 일부 사용자에게만 영향을 미칩니다.
3. **테스트 결과가 긍정적이면**, 다음 단계로 사용자 비율을 늘립니다 (예: 50%, 75%, 100% 순으로).
4. 각 단계에서 시스템을 다시 테스트하고, 문제가 발생하면 이전 버전으로 롤백할 수 있습니다.

![canary](image/canary.png)

### 3. 장점

- 문제가 발생할 경우 영향을 받는 사용자가 제한적이므로, 전체 시스템에 미치는 위험을 줄일 수 있습니다.
- 사용자 그룹을 선택하여 업데이트를 받을 대상을 조정할 수 있으며, 인구 통계, 물리적 위치, 디바이스 유형 등을 기준으로 사용자 선택이 가능합니다.
- Blue-Green 배포에 비해 환경을 복제할 필요가 없으므로 필요한 자원이 적습니다.

### 4. 단점

- 하지만 데이터베이스에 큰 변경이 있을 경우, 여전히 Blue-Green 배포와 유사한 문제에 직면할 수 있습니다.
- Canary 배포는 업데이트를 받을 사용자 그룹을 제한해야 하므로, 단순한 트래픽 전환만으로 이루어지는 Blue-Green 배포보다 구현이 더 복잡할 수 있습니다.


# 3. 스크립트를 활용한 무중단 배포

## Rolling 무중단 배포
우아한테크코스에서 진행하는 모우다는 기존에 스크립트를 이용해 서버를 배포했습니다. 
해당 환경을 유지하면서 무중단 배포를 도입하기 위해 스크립트 기반의 Rolling 무중단 배포를 진행하겠습니다.

```yaml
jobs:
  deploy-prod1: # 서버1 배포 
    name: Deploy to Prod1 Instance
    steps:
      - name: Run Prod1 instance deploy script
        run: |
          ...

  check-prod1: # 서버1 health-check
    name: Check Prod1 Instance
    needs: deploy-prod1

    steps:
      - name: Health check for Prod1 instance
        run: |
          ...

  deploy-prod2: # 서버2 배포
    name: Deploy to Prod2 Instance
    needs: check-prod1
    steps:
      - name: Run Prod2 instance deploy script
        run: |
          ...
          
  check-prod2: # 서버2 health-check
    name: Check Prod2 Instance
    needs: deploy-prod2
    steps:
      - name: Health check for Prod2 instance
        run: |
          ...
```
1. 첫 번째 서버(prod1)에 먼저 새 버전을 배포합니다.
2. 배포가 완료되면 health-check할 엔드포인트로 요청을 보내 기대한 응답코드를 반환하는지 확인합니다.
3. 첫 번째 서버(prod1)에서 응답을 잘 받으면 다음 서버(prod2)에 새로운 버전을 배포합니다.
4. prod2도 배포 완료 후 health-check를 진행합니다.


## 주의 사항

서버가 버전업 하는 동안 로드밸런서가 트래픽을 보내지 않도록 해야합니다.
저의 경우 ELB를 사용하기 때문에 로드밸런서가 health-check하는 간격을 최소화하여 배포하는 경우 가능한 빠르게 트래픽을 막을 수 있도록 조치했습니다.

![loadbalancer-configuration](image/loadbalancer-configuration.png)
- **제한 시간**: 2초 (각 health-check 요청의 최대 대기 시간)
- **비정상 임계값**: 3회 (비정상 상태로 간주하기 위해 연속 실패해야 하는 횟수)
- **간격**: 5초 (health-check 요청 사이의 대기 시간)

따라서 서버를 내리고 최소 **28초** 후에 로드밸런서가 비정상 상태로 인지하여 트래픽을 막습니다.

> ✔️ 28초 = 첫 요청 제한 (2초) + { 대기(5초) + 요청 제한(2초) } * 4회

약 28초 동안 요청을 제대로 처리하지 못할 수 있습니다.

즉, 아래처럼 서버를 종료해도 약 30초간 정상으로 판단하고 로드밸런서가 트래픽을 계속 보내는 것입니다.

![stop-server](image/stop-server.png)

![stop-traffic](image/stop-traffic.png)

## 개선 방법

1. 서버를 내리기 전에 로드밸런서에게 배포할 서버가 unhealthy 하다고 알림
2. 로드밸런서가 트래픽을 더이상 보내지 않을 때 서버 다운
3. 새 버전 배포
4. health-check

위 방식으로 임의로 health-check에 잘못된 응답을 보내 로드밸런서가 트래픽을 차단하도록 준비시킵니다.
로드밸런서가 비정상임을 인지하는 약 30초동안 서버는 요청을 처리할 수 있습니다.


**1. 서버를 내리기 전에 로드밸런서에게 배포할 서버가 unhealthy 하다고 알림**

```java
@RestController
public class HealthCheckController {

	private static final String HOST_IPV4 = "127.0.0.1";
	private static final String HOST_IPV6 = "0:0:0:0:0:0:0:1";
	private static final String HOST_NAME = "localhost";

	private final AtomicBoolean isTerminating = new AtomicBoolean(false);

	@GetMapping("/health")
	public ResponseEntity<Void> checkHealth() {
		if (isTerminating.get()) {
			return ResponseEntity.status(HttpStatus.BAD_GATEWAY).build();
		}
		return ResponseEntity.ok().build();
	}

	@PostMapping("/termination")
	public ResponseEntity<Void> terminate(HttpServletRequest request) {
		String remoteHost = request.getRemoteHost();
		if (HOST_IPV6.equals(remoteHost) || HOST_IPV4.equals(remoteHost) || HOST_NAME.equals(remoteHost)) {
			isTerminating.set(true);
			return ResponseEntity.ok().build();
		}
		return ResponseEntity.status(HttpStatus.FORBIDDEN).build();
	}
}

```

무중단 배포를 진행하기 전, 로드밸런서 health-check에서 `200`을 반환하지 않도록 만드는 `/termination` API를 하나 만들었습니다. 동시성 문제가 발생할 우려가 있어 `AtomicBoolean`을 사용합니다.

외부에서 `/termination` 에 요청을 방지하기 위해 `localhost`요청만 허용하도록 하겠습니다.

**2. 로드밸런서가 트래픽을 더이상 보내지 않을 때 서버 다운**

```yaml

jobs:
  deploy-prod1: # 서버1 배포 
    name: Deploy to Prod1 Instance
    steps:
      - name: Prepare Deploy
      - run: |
          curl -X POST http://localhost:8080/termination
          sleep 30
        
      - name: Run Prod1 instance deploy script
        run: |
          ...
```

health-check API가 `502`를 반환하도록 변경합니다. (POST `/termination`)

로드밸런서가 인지할 때까지 30초 대기하겠습니다.

이후 배포를 진행합니다.

**3. 새 버전 배포**
트래픽을 중단한 서버에 새로운 버전을 배포합니다.

**4. health-check**
새로운 버전 배포가 완료되었다면 health-check로 서버가 정상적으로 동작하는지 확인합니다.


지금까지 무중단 배포의 대표적인 방식 3가지와 장단점, 스크립트로 무중단 배포를 진행하는 방법을 작성하였습니다.
