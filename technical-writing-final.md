# DB는 롤백됐는데 S3 이미지는 남았다: 트랜잭션 결과에 S3 작업을 맞추며 겪은 문제 2가지

가치가챠는 가챠샵(캡슐 토이 매장)의 위치를 지도에서 찾고, 사용자끼리 중복된 가챠를 교환할 수 있는 서비스입니다.

처음에는 지도에서 가챠샵을 보여 주는 서비스로 시작했습니다. 하지만 홍대 가챠샵을 직접 방문해 스티커 설문을 해 보니, 지도 서비스보다 원하지 않는 가챠가 나왔을 때 교환할 곳이 필요하다는 응답이 많았습니다. 그래서 사용자끼리 중복된 가챠를 바꿀 수 있는 교환 게시글 기능을 추가했고, 현재 이 기능이 가치가챠의 주요 기능입니다.

가챠는 한 기계에서도 여러 종류의 상품이 무작위로 나오기 때문에, 내가 어떤 상품을 가졌는지 글만으로는 알기 어렵습니다. 따라서 교환 게시글에 상품 이미지를 올리는 기능을 구현했습니다.

이미지가 들어간 게시글은 아래 순서로 저장됩니다.

1. 사용자가 이미지와 함께 게시글 등록을 요청합니다.
2. 서버가 이미지를 Amazon S3(Simple Storage Service, 이하 S3)라는 파일 저장소에 올리고, 이미지의 주소(URL)를 받습니다.
3. 서버가 게시글 내용과 이미지 주소를 PostgreSQL DB(데이터베이스)에 저장합니다.
4. DB 저장이 모두 성공하면 확정(커밋)하고, 하나라도 실패하면 없던 일로 되돌립니다(롤백).

그런데 4번에서 되돌릴 수 있는 범위는 **DB뿐**입니다. 2번에서 S3에 올린 이미지는 그대로 남아, 어떤 게시글과도 연결되지 않은 이미지가 됩니다. 이 글에서는 이런 이미지를 **고아 파일**이라고 부릅니다.

이 문제를 막으려고 S3 작업을 DB 작업의 결과가 정해진 뒤로 미뤘습니다. 이 방법은 이미 여러 글에서 소개하고 있지만, 실제 코드에 적용해 보니 놓치기 쉬운 문제가 두 가지 있었습니다.

- **커밋 단계의 실패를 잡지 못하는 try-catch**: 올린 이미지를 catch 블록에서 지우게 했지만, 메서드가 끝난 뒤 커밋하다 실패하면 catch가 실행되지 않아 고아 파일이 남았습니다. (3장)
- **커밋 뒤 500 응답을 만드는 afterCommit 예외**: 커밋 뒤에 기존 이미지를 지우는 코드에서 예상하지 못한 예외가 나면, 그 예외가 API 응답까지 올라가 서버 오류(HTTP 500)가 됐습니다. 데이터는 이미 저장됐는데 사용자는 요청이 실패했다고 받는 문제였습니다. (4장)

두 문제 모두 방향은 맞았지만 세부 구현에서 생겼습니다. 같은 구조를 적용하려는 분들이 같은 지점에서 막히지 않도록 원인과 해결 방법을 정리했습니다.

**다루는 내용**

- DB 트랜잭션이 S3 작업을 되돌리지 못해서 생기는 세 가지 실패 (1장)
- Spring의 TransactionSynchronizationManager로 S3 작업을 트랜잭션 결과에 맞춘 구조 (2장)
- 두 문제의 원인과 해결 (3~4장)
- 검증 범위, 다른 외부 저장소에 적용하는 기준, 남은 과제 (5~7장)

**대상 독자**

트랜잭션을 막 사용하기 시작한 백엔드 개발자부터, 외부 저장소와 DB의 데이터를 일치시키는 문제를 고민하는 경력 개발자까지 염두에 두고 작성했습니다. 개발자가 아니라면 1장과 8장만 읽어도 어떤 문제를 어떻게 풀었는지 알 수 있습니다.

---

## 1. @Transactional의 경계

트랜잭션은 여러 DB 작업을 하나로 묶어서, 모두 반영하거나(커밋) 모두 없던 일로 되돌리는(롤백) 장치입니다. Spring에서는 메서드에 @Transactional을 붙이면 그 메서드 안의 DB 작업이 하나의 트랜잭션으로 묶입니다.

다만 트랜잭션이 되돌릴 수 있는 범위는 **DB 안의 작업뿐**입니다. S3에 이미지를 올리거나 지우는 일은 네트워크 너머의 다른 시스템에 보내는 요청이라, DB가 롤백되어도 S3에서 이미 일어난 일은 취소되지 않습니다. 이 차이 때문에 세 가지 실패가 생길 수 있었습니다.

1. 등록: S3에 이미지를 올린 뒤 DB 저장이 실패해 롤백되면, 고아 파일이 S3에 남습니다.
2. 수정: 기존 이미지를 먼저 지웠는데 트랜잭션이 롤백되면, DB는 기존 이미지 주소를 그대로 가지고 있지만 S3에는 그 이미지가 없습니다.
3. 삭제: DB에서 게시글을 지우기 전에 S3 이미지부터 지우거나 옮겼는데 롤백되면, 게시글은 남아 있고 이미지만 사라집니다.

![그림 1. 적용 전 실패 시나리오](./img/technical-writing/figure-1.png)

*그림 1. 적용 전 실패 시나리오. 실선은 트랜잭션 안의 DB 작업, 점선은 트랜잭션 밖의 S3 요청입니다.*

실제로 처음 작성한 가챠 삭제 코드도 S3 이미지를 먼저 휴지통 경로로 옮긴 뒤 DB 데이터를 지우는 3번 순서였습니다. 처음 기능을 구현할 때는 이 문제를 놓쳤고, 이후 코드를 다시 검토하는 과정에서 발견해 고쳤습니다.

세 실패는 무게가 다릅니다. 고아 파일은 저장 공간만 차지할 뿐 사용자 눈에는 보이지 않습니다. 반면 수정과 삭제에서 생기는 문제는 화면에 이미지가 뜨지 않는 장애로 바로 드러납니다. 이 차이는 2장에서 작업마다 다른 처리 시점을 고르는 기준이 됩니다.

---

## 2. 해결 방향: 트랜잭션 결과에 S3 작업 맞추기

### 일곱 가지 선택지와 판단 기준

해결 방법을 고르기 전에 판단 기준을 세 가지 정했습니다.

1. 실패한 뒤에 되돌리는 방식보다, 되돌릴 일이 처음부터 생기지 않는 방식을 먼저 봅니다.
2. 매장, 가챠, 교환 게시글이 같은 문제를 겪으니 같은 방식으로 풉니다.
3. 다른 팀원이 바로 가져다 사용할 수 있을 만큼 구현과 재사용 비용이 낮아야 합니다.

여기에 아직 일어나지 않은 문제를 위해 기술부터 도입하지 않는다는 원칙을 더했습니다. 이 원칙에 따라 비동기 처리(@Async)와 Outbox 패턴(실행할 작업을 DB에 기록해 두고 별도로 실행·재시도하는 방식)은 보류했습니다.

| 선택지 | 장점 | 단점 | 결과 |
| --- | --- | --- | --- |
| 위험 수용 | 추가 구현 없음 | 고아 파일과 깨진 이미지가 쌓임 | 제외 |
| 수동 try-catch | 익숙하고 단순함 | 커밋 단계의 실패를 못 잡고, 원래 예외를 가릴 수 있음 | 처음 시도 후 제거 (3장) |
| TransactionSynchronizationManager | 트랜잭션 결과를 확인한 뒤 실행, 추가 인프라 없음 | 서비스 코드가 Spring 트랜잭션 API에 의존 | 채택 |
| @TransactionalEventListener | 이벤트로 분리해 결합도가 낮음 | 이벤트·리스너 클래스가 늘고 흐름을 따라가기 어려움 | 후속 과제로 보류 |
| @Async | 응답 지연을 줄임 | 스레드 풀 관리가 필요하고, 아직 지연 문제가 없음 | 보류 |
| Outbox + 스케줄러 | 실패 시 재시도 가능 | 테이블과 스케줄러 추가, 현재 규모에 과함 | 보류 |
| S3 Lifecycle Rule(일정 기간이 지난 객체를 자동으로 지우는 S3 설정) | 코드 변경 없이 오래된 이미지 정리 | 고아 여부를 알 수 없고, 수정·삭제 문제는 못 막음 | 제외 |

*표 1. 검토한 선택지와 결과*

마지막까지 TransactionSynchronizationManager와 @TransactionalEventListener를 비교했습니다. @TransactionalEventListener도 내부에서는 같은 트랜잭션 동기화 콜백을 사용하지만, S3 작업이 리스너 클래스로 나뉘어서 서비스 코드만 읽어서는 롤백될 때 무엇이 정리되는지 알기 어렵습니다. 그래서 "이 트랜잭션이 롤백되면 이 이미지를 지운다"는 내용이 서비스 메서드에 그대로 보이는 TransactionSynchronizationManager를 골랐습니다. 다만 Spring 문서에 따르면 @TransactionalEventListener의 기본 단계인 `AFTER_COMMIT`은 afterCommit이 아니라 afterCompletion 콜백과 같은 흐름에서 실행됩니다. 나중에 이벤트 방식으로 옮긴다면 4장에서 다룰 예외 처리 차이를 다시 확인해야 합니다.

### 트랜잭션 동기화 콜백과 선택 규칙

Spring은 현재 트랜잭션이 끝나는 시점에 실행할 코드를 미리 맡겨 둘 수 있게 해 줍니다. 이렇게 맡겨 두는 코드를 콜백이라고 합니다. `TransactionSynchronizationManager.registerSynchronization()`에 `TransactionSynchronization` 구현체를 넘기면, Spring이 트랜잭션을 끝내면서 그 메서드를 호출합니다. 이 글에서 사용하는 콜백은 두 가지입니다.

- afterCommit(): 커밋이 성공했을 때만 호출됩니다.
- afterCompletion(int status): 커밋이든 롤백이든 트랜잭션이 끝나면 항상 호출되고, 결과를 `STATUS_COMMITTED`나 `STATUS_ROLLED_BACK` 같은 값으로 알려 줍니다.

두 콜백 모두 메서드가 반환되고 커밋까지 시도한 다음에 실행됩니다. 그래서 커밋 단계에서 실패해 롤백되더라도, afterCompletion은 그 실패까지 **"롤백됐다"는 결과로** 받습니다.

![그림 2. 트랜잭션이 끝날 때 콜백이 호출되는 순서](./img/technical-writing/figure-2.png)

*그림 2. 트랜잭션이 끝날 때 콜백이 호출되는 순서. afterCompletion은 어느 쪽으로 끝나든 호출됩니다.*

어떤 콜백을 사용할지는 1장에서 본 실패의 무게 차이로 정했습니다.

- 새로 올리는 작업은 먼저 실행하고, 롤백되면 **afterCompletion**에서 지웁니다. (3장)
- 기존 이미지를 지우는 작업은 커밋이 확정된 뒤 **afterCommit**에서 실행합니다. (4장)

지우는 작업을 커밋 뒤로 미루면, 롤백됐을 때 기존 이미지가 그대로 남아 화면에 이미지가 뜨지 않는 무거운 실패를 막을 수 있습니다. 대신 커밋 뒤에 삭제가 실패하면 사용자에게 보이지 않는 고아 파일이 남습니다.

매장과 가챠의 이미지 서비스, 매장과 가챠의 삭제 서비스까지 네 곳에 비슷한 콜백 코드가 반복돼서, 이 규칙을 `S3TransactionManager`라는 컴포넌트 하나로 모았습니다(커밋 `3e0f44b`). 이후 교환 게시글 서비스도 같은 컴포넌트를 사용합니다.

| 메서드 | 콜백 | 동작 | 사용하는 곳 |
| --- | --- | --- | --- |
| `deleteImagesOnRollback` | afterCompletion | 롤백되면 올린 이미지를 삭제, 커밋되면 아무것도 하지 않음 | 이미지 등록 |
| `trashImagesAfterRemoved` | afterCommit | 커밋된 뒤에만 이미지를 `trash/` 경로로 이동 | 가챠·교환 게시글 삭제, 교체 시 기존 이미지 |
| `deleteImagesAfterRemoved` | afterCommit | 커밋된 뒤에만 이미지를 완전히 삭제 | 매장 삭제 |
| `cleanupAfterImageReplaced` | afterCompletion | 커밋이면 기존 이미지를 휴지통으로, 롤백이면 새 이미지를 삭제 | 이미지 1장 교체 |

*표 2. S3TransactionManager의 메서드*

---

## 3. 첫 번째 문제: 커밋 단계의 실패를 잡지 못하는 try-catch

### 상황: try-catch로 정리한 등록 코드

삭제와 교체는 먼저 트랜잭션 동기화 콜백으로 옮겼지만(커밋 `984034d`), 이미지 등록은 한동안 익숙한 방법으로 처리했습니다. 올린 이미지의 주소를 목록에 모아 두고, 예외가 나면 catch 블록에서 그 이미지들을 지운 뒤 예외를 다시 던지는 방식입니다.

```java
// GachaImageService (수정 전, 일부)
@Transactional
public List<GachaImageInfo> addImage(final Long gachaId, final List<MultipartFile> files) {
    List<String> uploadedImageUrls = new ArrayList<>();
    try {
        // 1. 이미지를 하나씩 S3에 올리고, 성공한 주소를 uploadedImageUrls에 추가
        // 2. 올린 주소로 GachaImage를 만들어 saveAll()로 저장
        ...
    } catch (RuntimeException e) {
        uploadedImageUrls.forEach(imageUploader::delete);  // 올린 이미지 정리
        throw e;
    }
}
```

### 원인: catch가 닿지 않는 커밋 단계

빈틈이 없어 보이지만 이 catch가 잡지 못하는 실패가 있습니다. Spring Boot에서 JPA(Java Persistence API, 자바 객체를 DB에 저장하는 표준 기술)를 사용하면 기본 구현체로 Hibernate가 동작합니다. Hibernate는 엔티티의 변경 내용을 모아 두었다가 정해진 시점에 SQL(DB에 보내는 명령문)로 바꿔 DB에 보내는데, Hibernate 문서는 이 작업을 flush라고 부릅니다. 기본 설정에서는 커밋 직전에도 flush가 일어나서, 제약조건 위반 같은 예외가 이때 날 수 있습니다. 커밋 자체가 실패하는 경우도 마찬가지입니다.

실행 순서를 따라가 보면 catch가 왜 닿지 않는지 보입니다.

1. 서비스 메서드 본문이 실행됩니다. try 블록이 지켜보는 구간은 여기까지입니다.
2. 메서드가 정상적으로 반환됩니다.
3. Spring이 커밋을 시작하고, 커밋 직전 flush에서 SQL이 실행됩니다.
4. 이 단계에서 실패하면 트랜잭션이 롤백됩니다.

4번의 실패가 일어날 때 실행 흐름은 **이미 catch 블록을 빠져나온 뒤**입니다. 그래서 DB는 롤백되는데 올린 이미지는 S3에 그대로 남습니다.

![그림 3. 수동 try-catch가 닿지 않는 구간](./img/technical-writing/figure-3.png)

*그림 3. 수동 try-catch가 닿지 않는 구간. 회색 상자 밖에서 난 실패는 catch로 잡을 수 없습니다.*

문제가 하나 더 있었습니다. catch 안의 `imageUploader::delete`도 실패하면 예외를 던집니다. 그러면 정리 중에 난 예외가 원래 예외를 가려서, 로그에는 S3 삭제 실패만 남고 정작 저장이 왜 실패했는지는 사라집니다.

### 해결: 롤백 결과에 반응하는 afterCompletion 콜백

실패가 어디서 났든 "트랜잭션이 롤백됐다"는 결과에 반응하는 장치가 필요했고, 2장에서 본 afterCompletion이 그 역할을 합니다. 그래서 등록 코드의 catch를 걷어내고 `deleteImagesOnRollback()`에 정리를 맡겼습니다. 이 메서드는 콜백을 등록만 하고, 실제 S3 작업은 트랜잭션이 끝난 뒤에 실행됩니다.

```java
// S3TransactionManager (일부)
public void deleteImagesOnRollback(
        final DomainType imageType,
        final Long domainId,
        final List<String> imageUrls
) {
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCompletion(final int status) {
            if (status == TransactionSynchronization.STATUS_ROLLED_BACK) {
                imageUrls.forEach(imageUrl -> deleteImage(imageType, domainId, imageUrl));
            }
        }
    });
}
```

### 신경 쓸 점: 콜백 등록 위치

앞의 try-catch에도 장점이 하나 있었습니다. 업로드에 성공할 때마다 목록에 주소를 쌓았기 때문에, 세 장을 올리다 두 번째에서 실패해도 첫 번째 이미지를 지울 수 있었습니다. 업로드를 모두 마친 뒤에 콜백을 등록하면 이 장점을 잃습니다. 두 번째 업로드에서 예외가 나면 실행 흐름이 등록 코드까지 가지 못해, 첫 번째 이미지가 S3에 남기 때문입니다.

그래서 **업로드 전에 빈 목록으로 콜백을 먼저 등록**하고, 업로드에 성공할 때마다 같은 목록에 주소를 쌓았습니다. 콜백은 목록의 복사본이 아니라 같은 목록 객체를 가리키므로, 롤백 시점에는 그때까지 올라간 이미지를 모두 정리합니다.

![그림 4. 콜백 등록 위치에 따른 차이](./img/technical-writing/figure-4.png)

*그림 4. 콜백 등록 위치에 따른 차이. 등록이 늦으면 실패 직전까지 올라간 이미지가 정리 대상에서 빠집니다.*

```java
// TradeService (일부)
private List<String> uploadImages(final Trade trade, final List<MultipartFile> images) {
    if (images == null || images.isEmpty()) {
        return List.of();
    }

    List<String> uploadedUrls = new ArrayList<>();
    s3TransactionManager.deleteImagesOnRollback(DomainType.TRADE, trade.getId(), uploadedUrls);  // 업로드 전에 등록

    for (MultipartFile image : images) {
        String imageUrl = multipartUploader.upload(image, DomainType.TRADE);
        uploadedUrls.add(imageUrl);  // 콜백이 바라보는 목록에 누적
        tradeImageRepository.save(TradeImage.builder().trade(trade).imageUrl(imageUrl).build());
    }

    return uploadedUrls;
}
```

이 변경을 담은 커밋(`53fb6ef`) 메시지에는 "실패 시점과 무관하게 롤백되면 항상 정리되도록"이라고 적었습니다. 메서드 본문에서 실패하든, 업로드 도중이나 커밋 단계에서 실패하든 결과가 롤백이면 같은 콜백이 정리합니다.

---

## 4. 두 번째 문제: 커밋 뒤 500 응답을 만드는 afterCommit 예외

### 상황: afterCommit으로 옮긴 삭제와 좁아진 catch

기존 이미지를 지우는 작업은 2장의 규칙대로 afterCommit에서 실행합니다. 교환 게시글처럼 사용자가 올린 콘텐츠는 신고에 대응할 때 원본이 필요할 수 있어서 바로 지우지 않고 `trash/` 경로로 옮기고, 매장 이미지는 완전히 지웁니다. S3에는 객체를 한 번에 옮기는 기능이 없어서, 휴지통 이동은 AWS(Amazon Web Services) 문서가 안내하는 대로 새 위치로 복사한 뒤 원본을 지우는 방식으로 구현했습니다.

처음 삭제 작업을 afterCommit 콜백으로 옮길 때(커밋 `984034d`), S3 정리 메서드의 catch 범위도 `RuntimeException`에서 `S3Exception`으로 좁혔습니다.

```diff
// GachaService (커밋 984034d)
 private void moveFileToTrash(final Long gachaId, final GachaImage gachaImage) {
     try {
         imageUploader.moveToTrash(gachaImage.getImageUrl());
-    } catch (RuntimeException e) {
+    } catch (S3Exception e) {
         log.warn("가챠 삭제 중 S3 이미지 삭제에 실패했습니다. gachaId={}, imageUrl={}", gachaId, gachaImage.getImageUrl(), e);
     }
 }
```

`S3Exception`은 프로젝트에서 직접 만든 예외로, AWS SDK(Software Development Kit, 자바 코드에서 AWS를 사용하게 해 주는 공식 라이브러리)가 던지는 `SdkException`을 감싸서 던집니다. 그래서 S3와 통신하다 난 실패는 이 catch에 잡힙니다. 하지만 S3 호출 앞뒤의 코드에서 나는 예외는 빠져나갑니다. 예를 들어 휴지통 경로를 만드는 `generateTrashKey()`는 이미지 키에 `/`가 없으면 `indexOf('/')`가 -1을 돌려주고, 이어지는 `substring(0, -1)`에서 `StringIndexOutOfBoundsException`을 던집니다.

당시 실제로 어떤 예외가 났는지는 기록에 남아 있지 않지만, 이런 예외가 afterCommit 콜백 밖으로 나가면 DB는 이미 커밋됐는데 API가 500을 반환합니다.

### 원인: 콜백별 예외 처리 차이

같은 예외가 나도 afterCommit과 afterCompletion은 다르게 처리합니다. Spring의 `TransactionSynchronization` 문서가 이를 명시합니다.

| 콜백 | 예외가 나면 | 문서 표현 |
| --- | --- | --- |
| afterCommit() | 호출자에게 전파됨 | "will be propagated to the caller" |
| afterCompletion(int) | 로그만 남고 전파되지 않음 | "will be logged but not propagated" |

*표 3. 콜백별 예외 처리 방식*

커밋이 이미 끝난 트랜잭션은 되돌릴 수 없습니다. 그런데 afterCommit에서 난 예외는 **호출자에게 그대로 전달**되니, 결과적으로 "데이터는 저장됐고 호출자는 예외를 받는" 상태가 됩니다.

![그림 5. afterCommit 예외가 응답까지 올라가는 경로](./img/technical-writing/figure-5.png)

*그림 5. afterCommit 예외가 응답까지 올라가는 경로*

### 해결: 콜백 안에서 모든 예외 잡기

커밋 `9559e9c`에서 두 가지를 바꿨습니다.

1. catch 범위를 다시 `RuntimeException`으로 넓혔습니다. afterCommit 시점에는 데이터가 이미 저장됐으므로, 클라이언트에 돌려줄 올바른 응답은 성공입니다.
2. 로그 레벨을 WARN에서 ERROR로 올렸습니다. 정리에 실패하면 고아 파일이 생기고, 저절로 해결되지 않아 사람이 확인해야 하기 때문입니다.

```diff
// S3TransactionManager (커밋 9559e9c, 타입 이름은 현재 코드 기준)
 private void moveImageToTrash(final DomainType imageType, final Long domainId, final String imageUrl) {
     try {
         imageUploader.moveToTrash(imageUrl);
-    } catch (final S3Exception e) {
-        log.warn("{} 이미지를 휴지통으로 이동하는 데 실패했습니다. domainId={}, imageUrl={}",
+    } catch (final RuntimeException e) {
+        log.error("{} 이미지를 휴지통으로 이동하는 데 실패했습니다. domainId={}, imageUrl={}",
                 imageType.getLabel(), domainId, imageUrl, e);
     }
 }
```

같은 커밋에서 완전 삭제를 담당하는 `deleteImage()`도 똑같이 바꿨습니다.

### 신경 쓸 점: 커밋 뒤에 실행하는 코드의 책임

afterCommit 안의 코드는 **예외를 밖으로 내보내지 않도록** 작성해야 합니다. 실패를 클라이언트에 알려야 하는 작업이라면 커밋 뒤가 아니라 커밋 전에 실행해야 합니다. 반대로 커밋 뒤로 미룬 작업은 실패해도 응답을 바꾸지 않는 대신, ERROR 로그처럼 사람이 알아챌 수 있는 흔적을 남겨야 합니다.

이미지 교체처럼 두 작업이 한 요청에 함께 들어 있다면, 기존 이미지는 afterCommit에, 새 이미지는 afterCompletion에 각각 맡기면 됩니다. 어느 쪽으로 끝나든 DB와 S3가 같은 상태를 가리킵니다.

---

## 5. 검증 범위와 한계

`S3TransactionManagerTest`(커밋 `3962779`)에서 네 메서드가 다음처럼 동작하는지 확인했습니다.

- 커밋되면 기존 이미지는 휴지통으로 옮기거나 삭제하고, 새로 올린 이미지는 그대로 둡니다.
- 롤백되면 새로 올린 이미지만 삭제하고, 기존 이미지는 건드리지 않습니다.
- 콜백 안에서 S3 작업이 `RuntimeException`을 던져도 그 예외를 밖으로 내보내지 않습니다.

3장의 "목록을 참조로 넘기는" 설계도, 콜백을 등록한 뒤에 목록을 채우고 롤백 시점에 채워진 주소가 모두 삭제되는지로 확인했습니다.

다만 이 테스트에는 한계가 있습니다. S3와 통신하는 객체는 가짜 객체(mock)로 바꿨고, 콜백도 실제 트랜잭션 매니저가 아니라 테스트 코드에서 afterCommit()과 afterCompletion()을 직접 호출해 실행했습니다. 그래서 실제 S3 환경의 네트워크 오류나 타임아웃, 그리고 실제 트랜잭션에서 커밋 단계 실패가 일어났을 때 afterCompletion이 끝까지 정리하는지는 아직 실험으로 확인하지 못했습니다. 두 번째는 Spring과 Hibernate 문서, 그리고 코드 흐름을 근거로 한 추론입니다.

같은 문제를 검증하려는 분께는 두 가지 방법을 권합니다. 첫째, Testcontainers(테스트할 때 도커 컨테이너를 띄워 주는 라이브러리)의 LocalStack 모듈로 S3 호환 환경을 띄우고, 실제 트랜잭션 안에서 롤백을 일으켜 버킷의 객체 목록을 비교하는 통합 테스트를 만드는 방법입니다. 둘째, 운영 버킷의 객체 목록과 DB의 이미지 주소를 대조해 실제로 쌓인 고아 파일 수를 세는 방법입니다.

---

## 6. 트랜잭션이 없는 경로와 적용 기준

트랜잭션이 없는 경로에는 이 구조를 적용할 수 없습니다. 채팅 파일 업로드 API(`POST /files`)는 파일만 올리고 DB에는 아무것도 저장하지 않아 트랜잭션이 없습니다. Spring 문서에 따르면 트랜잭션 동기화가 비활성 상태에서 콜백을 등록하면 `IllegalStateException`이 나므로, 여기서는 여러 파일을 올리다 중간에 실패하면 catch에서 앞서 올린 파일을 지우게 했습니다. 트랜잭션이 없으니 3장의 커밋 단계 사각지대도 없습니다. 파일만 올리고 메시지는 보내지 않아서 생기는 고아 파일은 S3 Lifecycle Rule로 정리하기로 하고 보류했습니다.

이 경험을 바탕으로, 트랜잭션 밖의 저장소를 다루는 작업에 적용할 기준을 정리했습니다.

| 상황 | 처리 방법 | 실패 시 남는 것 |
| --- | --- | --- |
| 트랜잭션 안에서 외부에 새로 만듦 | 실행 전에 afterCompletion 콜백 등록, 롤백이면 삭제 | 정리 실패 시 고아 데이터 |
| 트랜잭션 안에서 외부의 기존 것을 지우거나 바꿈 | afterCommit에서 실행 | 실행 실패 시 고아 데이터 |
| 교체 (새로 만들고 기존 것을 지움) | 위 두 가지를 함께 등록 | 위와 같음 |
| 트랜잭션 없음 | 호출한 쪽에서 직접 정리, 남는 것은 주기적으로 정리 | 정리 누락분 |
| 반드시 성공해야 함 (재시도 필요) | Outbox 패턴 같은 재시도 구조 | 지연은 있어도 유실 없음 |

*표 4. 외부 저장소 작업 유형별 처리 기준*

어떤 방법을 사용해도 실패를 완전히 없앨 수는 없고, 실패했을 때 **무엇을 남길지만** 고를 수 있습니다. 이 프로젝트에서는 사용자 눈에 보이는 깨진 이미지 대신, 보이지 않는 고아 파일이 남는 쪽을 골랐습니다.

---

## 7. 남은 과제

지금 구조에도 알고 있는 한계가 네 가지 있습니다. 아직 실제 문제로 관측되지 않아서, 2장의 원칙대로 보류해 두었습니다.

| 한계 | 영향 | 다음 단계 |
| --- | --- | --- |
| 업로드 중 DB 커넥션 점유 | S3가 느려지면 커넥션 풀이 바닥나 다른 API까지 실패 | 업로드를 트랜잭션 밖으로 이동, Presigned URL |
| 응답을 늦추는 afterCommit 작업 | S3 정리가 끝나야 응답이 나감 | 비동기 실행(@Async) 재검토 |
| 재시도 없는 정리 실패 | 고아 파일이 조용히 쌓임 | 정리 배치, Outbox 패턴 |
| 병렬 업로드와 스레드 경계 | 업로드 스레드에서 콜백 등록 불가 | 요청 스레드에서 등록하고 주소만 모으기 |

*표 5. 남은 과제*

### 업로드 중 DB 커넥션 점유

업로드가 트랜잭션 안에서 실행되어, S3에 이미지를 올리는 동안 DB 커넥션(서버와 DB 사이의 연결)을 붙잡고 있습니다. 서버는 커넥션 풀(DB 커넥션을 미리 만들어 두고 요청마다 빌려주는 장치)에서 커넥션을 빌려 쓰고, 이 프로젝트의 설정은 다음과 같습니다.

- S3 호출 제한 시간(apiCallTimeout): 60초. AWS 문서에 따르면 재시도를 포함한 전체 시간입니다.
- HikariCP 커넥션 풀 크기: 최대 10개
- 커넥션을 기다리는 최대 시간(connectionTimeout): 30초. HikariCP 문서에 따르면 이 시간이 지나면 예외가 납니다.

S3가 느려진 상태에서 업로드 요청 10건이 동시에 들어오면 커넥션이 바닥나, S3와 상관없이 DB를 사용하는 다른 API 요청까지 30초를 기다린 뒤 실패합니다(연쇄 장애).

![그림 6. 느린 S3 업로드가 커넥션 풀을 바닥내는 과정](./img/technical-writing/figure-6.png)

*그림 6. 느린 S3 업로드가 커넥션 풀을 바닥내는 과정*

업로드를 트랜잭션 밖으로 빼거나, Presigned URL(AWS 자격 증명 없이도 정해진 시간 동안 특정 객체를 올릴 수 있게 해 주는 서명된 주소)로 클라이언트가 S3에 직접 올리게 하면 이 문제를 줄일 수 있습니다. 다만 그때는 콜백을 걸 트랜잭션이 없으므로 정리 방식도 다시 설계해야 합니다. 우아한테크코스 기술 블로그 Tecoble의 「트랜잭션 내에 외부 리소스 요청이 담기게 되면 어떤 문제가 발생할까?」도 같은 문제를 다룹니다.

### 응답을 늦추는 afterCommit 작업

콜백은 요청을 처리하는 스레드에서 순서대로 실행되므로, afterCommit의 S3 작업이 끝나야 응답이 나갑니다. `TransactionSynchronization` 문서에 따르면 이 시점에 트랜잭션 자원이 아직 활성 상태일 수도 있습니다. 응답 시간이 문제가 되면 2장에서 보류한 비동기 실행을 다시 검토합니다.

### 재시도 없는 정리 실패

정리에 실패하면 ERROR 로그만 남고 재시도는 없습니다. 4장에서 예외를 밖으로 내보내지 않기로 한 대가입니다. 다음 단계는 버킷과 DB를 주기적으로 대조하는 정리 배치나 Outbox 패턴입니다. 우아한형제들 기술블로그의 「회원시스템 이벤트기반 아키텍처 구축하기」는 이벤트를 도메인 데이터와 같은 DB에 저장하고, 저장 후 5분이 지나도 발행되지 않은 이벤트를 배치가 다시 발행하는 구조를 소개합니다.

### 병렬 업로드와 스레드 경계

`TransactionSynchronizationManager`는 ThreadLocal(스레드마다 따로 값을 보관하는 자바 기능)을 기반으로 동작합니다. 그래서 `CompletableFuture`(작업을 다른 스레드에 맡겨 비동기로 실행하는 자바 기능)로 다른 스레드에서 업로드하면, 그 스레드에서는 콜백을 등록할 수 없습니다. 그림 7처럼 요청 스레드가 콜백을 먼저 등록하고, 업로드 스레드는 주소만 돌려주는 구조가 필요합니다. 아직 검토만 했고 구현하지 않았습니다.

![그림 7. 병렬 업로드에서 콜백을 등록하는 위치](./img/technical-writing/figure-7.png)

*그림 7. 병렬 업로드에서 콜백을 등록하는 위치*

Spring의 `TaskDecorator`(작업 실행 전후에 코드를 끼워 넣는 인터페이스)로 ThreadLocal 값을 새 스레드에 복사하는 방법도 있습니다. 다만 Spring 문서는 @Transactional의 트랜잭션이 메서드 안에서 새로 시작한 스레드로 전파되지 않는다고 설명합니다. 트랜잭션 정보까지 복사하면 DB 커넥션 하나를 여러 스레드가 함께 쓰게 될 수 있어서, 실험으로 확인해 볼 토의 거리로 남겨 둡니다.

---

## 8. 정리와 적용 체크리스트

DB 트랜잭션은 S3에서 일어난 일을 되돌리지 못합니다. 이 글에서는 S3 작업을 트랜잭션 결과에 맞추는 방법과, 그 과정에서 겪은 문제 두 가지를 다뤘습니다.

- 새로 올린 이미지: try-catch로 지우면 커밋 단계의 실패를 놓칩니다. 롤백되면 afterCompletion에서 지우고, 콜백은 S3 작업보다 먼저 등록해야 실패 시점과 상관없이 정리됩니다.
- 기존 이미지: 커밋이 확정된 뒤 afterCommit에서 지웁니다. 다만 afterCommit 안의 예외는 호출자에게 전파되므로, 이미 커밋된 요청을 500으로 만들지 않으려면 콜백 안에서 모든 예외를 잡아야 합니다.

처음에는 실패하면 어떻게 되돌릴지를 고민했지만, 되돌릴 일이 생기지 않는 순서로 바꾸는 편이 더 단순했습니다. 지우는 작업을 커밋 뒤로 미루자 롤백을 되돌리는 코드 자체가 필요 없어졌습니다. 다만 언제 등록하고 어떤 예외를 잡아야 하는지 같은 세부 사항은 직접 부딪히기 전까지 보이지 않았습니다. 같은 구조를 적용하려는 분이라면 아래 항목부터 확인해 보세요.

- [ ] 트랜잭션 안에서 S3, 다른 DB, 외부 API처럼 롤백되지 않는 저장소를 건드리나요?
- [ ] 새로 만드는 작업이라면, 실행 전에 롤백 콜백을 등록했나요?
- [ ] 기존 것을 지우는 작업이라면, afterCommit으로 미뤘나요?
- [ ] afterCommit 콜백 안의 예외를 종류와 상관없이 모두 잡고 있나요?
- [ ] 정리 실패가 ERROR 로그처럼 사람이 알아챌 수 있는 형태로 남나요?
- [ ] 트랜잭션이 없는 경로는 따로 처리했나요?

### 참고 자료

- Spring Framework, [TransactionSynchronization](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronization.html)
- Spring Framework, [TransactionSynchronizationManager](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronizationManager.html)
- Spring Framework, [TransactionPhase](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/event/TransactionPhase.html)
- Spring Framework, [Understanding the Spring Framework's Declarative Transaction Implementation](https://docs.spring.io/spring-framework/reference/data-access/transaction/declarative/tx-decl-explained.html)
- Spring Framework, [TaskDecorator](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/core/task/TaskDecorator.html)
- Hibernate ORM, [Flushing](https://docs.jboss.org/hibernate/orm/5.3/userguide/html_single/chapters/flushing/Flushing.html)
- HikariCP, [README: Configuration](https://github.com/brettwooldridge/HikariCP)
- AWS, [Handling errors in the AWS SDK for Java 2.x](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/handling-exceptions.html)
- AWS, [Configure timeouts in AWS SDK for Java 2.x](https://docs.aws.amazon.com/sdk-for-java/latest/developer-guide/timeouts.html)
- AWS, [Copying, moving, and renaming objects](https://docs.aws.amazon.com/AmazonS3/latest/userguide/copy-object.html)
- AWS, [Download and upload objects with presigned URLs](https://docs.aws.amazon.com/AmazonS3/latest/userguide/using-presigned-url.html)
- Testcontainers, [LocalStack Module](https://java.testcontainers.org/modules/localstack)
- 권용근, ["회원시스템 이벤트기반 아키텍처 구축하기"](https://techblog.woowahan.com/7835/), 우아한형제들 기술블로그, 2022.04
- 4기 매트, ["트랜잭션 내에 외부 리소스 요청이 담기게 되면 어떤 문제가 발생할까?"](https://tecoble.techcourse.co.kr/post/2022-09-20-external-in-transaction/), Tecoble, 2022.09.20
- 가치가챠 팀, [2026-gachi-gacha](https://github.com/woowacourse-teams/2026-gachi-gacha), GitHub (이 글에 인용한 코드와 커밋)
