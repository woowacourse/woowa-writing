# DB는 롤백됐는데, S3 이미지는 어디로 갔을까? 이미지 교체의 4가지 처리 경로

DAYNOMY에서 관리자 뉴스 기능을 개발하면서 겪은 이미지 정합성 문제를 소개하려고 합니다.

뉴스 이미지를 교체하는 작업은 처음엔 단순해 보였습니다. 새 이미지를 S3에 올리고, DB의 이미지 URL을 바꾸고, 기존 이미지를 지우면 끝입니다. 그런데 실패 경로를 따라가 보니 놓친 부분이 있었습니다. DB 커밋이 실패하면 URL은 이전 값으로 돌아가지만, S3에서 삭제한 이미지는 함께 돌아오지 않았습니다.

이 문제를 해결하기 위해 Spring의 `TransactionSynchronization`을 사용했습니다. 이번 글에서는 왜 `@Transactional`만으로 해결할 수 없었는지, `flush()` 전후의 실패를 어떻게 나눴는지, 커밋과 롤백에 따라 이미지 정리 시점을 어떻게 바꿨는지 이야기해 보겠습니다.

Spring과 자바 영속성 API(JPA)를 사용하면서 S3, 메일, 캐시처럼 DB 트랜잭션에 참여하지 않는 시스템을 함께 다루는 분께 도움이 되었으면 합니다.

## 시작은 단순한 이미지 교체였습니다

DAYNOMY는 뉴스 이미지 파일을 Amazon S3(이하 S3)에 저장하고, DB에는 이미지 URL을 기록합니다. 기존 이미지 교체 로직은 다음 순서로 동작했습니다.

```text
1. 새 이미지 업로드
2. DB의 이미지 URL 변경
3. 기존 이미지 삭제
4. DB 커밋
```

모든 작업이 성공하면 문제가 없습니다. 하지만 기존 이미지를 삭제한 뒤 DB 커밋에 실패하면 어떻게 될까요?

```text
1. 새 이미지 업로드 성공
2. DB의 이미지 URL 변경
3. 기존 이미지 삭제 성공
4. DB 커밋 실패
5. DB는 기존 URL로 롤백
6. 기존 URL을 조회하면 이미지가 나오지 않음
```

DB는 기존 URL로 돌아갑니다. S3 삭제 요청은 JPA 트랜잭션에 참여하지 않기 때문에 함께 롤백되지 않습니다. `@Transactional`이 보호하는 범위는 DB 트랜잭션이며, 외부 시스템에 이미 보낸 요청까지 되돌려 주지는 않습니다.

S3의 동작도 버킷 설정에 따라 조금 다릅니다. [AWS의 `DeleteObject` 문서](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html)에 따르면 버전 관리가 꺼진 버킷에서는 객체가 영구 삭제됩니다. 버전 관리가 켜진 버킷에서는 현재 버전에 삭제 마커가 생깁니다. 삭제 마커가 현재 버전이면 일반 조회는 `404 Not Found`를 반환합니다. 별도의 복구 절차가 없다면 사용자는 두 경우 모두 기존 이미지를 볼 수 없습니다.

여기서 한 가지 기준을 세웠습니다.

> DB가 현재 가리키는 이미지 파일은 조회할 수 있어야 한다.

이 기준으로 커밋과 롤백 상황을 나누면 해야 할 일이 선명해집니다.

| 트랜잭션 결과 | DB가 가리키는 이미지 | 남길 파일 | 정리할 파일 |
| --- | --- | --- | --- |
| 커밋 | 새 이미지 | 새 이미지 | 기존 이미지 |
| 롤백 | 기존 이미지 | 기존 이미지 | 새 이미지 |

DB 커밋이 확정되기 전에는 기존 이미지를 지우지 않습니다. 새 이미지는 URL을 얻기 위해 먼저 업로드하고, DB 반영에 실패하면 보상 작업으로 정리합니다.

## 커밋 결과를 기다렸다가 삭제하기

Spring의 `TransactionSynchronization`은 트랜잭션의 처리 단계에 맞춰 콜백을 실행할 수 있는 인터페이스입니다. DAYNOMY에서는 `afterCommit()`과 `afterCompletion()`을 사용했습니다.

```java
private void registerImageCleanup(
    String previousImageUrl, S3ImageStorage.StoredImage uploadedImage) {
  TransactionSynchronizationManager.registerSynchronization(
      new TransactionSynchronization() {
        @Override
        public void afterCommit() {
          deletePreviousImage(previousImageUrl);
        }

        @Override
        public void afterCompletion(int status) {
          if (status == STATUS_ROLLED_BACK) {
            deleteUploadedImage(uploadedImage);
          }
        }
      });
}
```

`registerSynchronization()`을 호출해도 삭제 작업은 바로 실행되지 않습니다. 현재 트랜잭션에 콜백을 등록해 두면 Spring 트랜잭션 관리자가 처리 결과에 맞는 콜백을 호출합니다. [Spring 공식 문서](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronization.html)에 따르면 `afterCommit()`은 주 트랜잭션이 성공적으로 커밋된 뒤 호출되고, `afterCompletion()`은 커밋이나 롤백이 끝난 뒤 호출됩니다.

커밋에 성공하면 DB는 새 이미지 URL을 가리킵니다. 이때 `afterCommit()`에서 기존 이미지를 삭제합니다. 롤백된 경우에는 `afterCommit()`이 호출되지 않고, `afterCompletion()`에 `STATUS_ROLLED_BACK`이 전달됩니다. 이번 요청에서 올렸지만 DB 어디에서도 참조하지 않는 새 이미지를 이 시점에 지웁니다.

```text
                           ┌─ 커밋 성공 ─→ 기존 이미지 삭제
새 이미지 업로드 → URL 변경 → 콜백 등록
                           └─ 롤백 ─────→ 새 이미지 삭제
```

콜백을 등록하려면 트랜잭션 동기화가 활성화되어 있어야 합니다. 이 로직은 `@Transactional` 서비스 메서드 안에서 호출하고 있습니다.

## `flush()` 전후를 4가지 경로로 나누기

콜백을 등록하기 전에 `flush()`를 호출하는 코드도 추가했습니다.

```java
S3ImageStorage.StoredImage uploadedImage = imageUpload.storedImage();
try {
  news.update(/* 새 이미지 URL을 포함한 변경 내용 */);

  if (uploadedImage != null) {
    newsRepository.flush();
    registerImageCleanup(previousImageUrl, uploadedImage);
  }

  return news;
} catch (RuntimeException exception) {
  deleteUploadedImage(uploadedImage);
  throw exception;
}
```

JPA는 영속성 컨텍스트의 변경 내용을 커밋 시점까지 DB에 보내지 않을 수 있습니다. [Jakarta Persistence 명세](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2.pdf)는 `flush()`가 영속성 컨텍스트와 DB의 동기화를 강제한다고 설명합니다. 제약조건 위반처럼 SQL 실행 중 확인되는 오류를 콜백 등록 전에 발견할 수 있습니다.

여기서 주의할 부분이 있습니다. `flush()`에 성공해도 커밋이 끝난 것은 아닙니다. SQL 실행 뒤에도 트랜잭션은 롤백될 수 있으므로 콜백이 여전히 필요합니다.

애플리케이션 프로세스가 정상적으로 실행 중인 상황을 기준으로 이미지 교체 흐름을 네 경로로 나눠 보았습니다.

| 경로 | 확인되는 시점 | 정리 주체 | 처리 결과 |
| --- | --- | --- | --- |
| S3 업로드 실패 | DB 변경 전 | 업로드 호출 | 요청 종료, 추가 파일 없음 |
| 업로드 성공 후 `flush()` 실패 | 콜백 등록 전 | `catch` | 새 이미지 삭제 |
| 콜백 등록 후 트랜잭션 롤백 | `afterCompletion()` | 롤백 콜백 | 새 이미지 삭제 |
| 트랜잭션 커밋 성공 | `afterCommit()` | 커밋 콜백 | 기존 이미지 삭제 |

`flush()` 도중 예외가 발생하면 아직 콜백이 등록되지 않은 상태입니다. 이때는 `catch` 블록에서 새 이미지를 바로 삭제합니다. 콜백 등록이 끝난 뒤 트랜잭션이 롤백되면 `afterCompletion()`이 새 이미지를 정리합니다. 각 구간의 정리 주체를 나눠 두니 실패 시점마다 어느 코드가 동작해야 하는지도 확인하기 쉬워졌습니다.

## 삭제했는지보다 언제 삭제했는지 테스트하기

이번 로직에서 중요하게 본 부분은 삭제 시점이었습니다. 삭제 메서드가 호출됐다는 사실만 확인하면 기존 이미지를 커밋 전에 지우는 문제를 놓칠 수 있습니다.

테스트에서는 트랜잭션 동기화를 직접 활성화하고, 서비스 호출 직후와 콜백 실행 후를 나눠 검증했습니다.

```java
TransactionSynchronizationManager.initSynchronization();
try {
  News updatedNews = adminNewsService.update(1L, request, image);

  verify(s3ImageStorage, never())
      .deleteIfManaged(previousImageUrl);

  for (TransactionSynchronization synchronization :
      TransactionSynchronizationManager.getSynchronizations()) {
    synchronization.afterCommit();
  }

  verify(s3ImageStorage).deleteIfManaged(previousImageUrl);
} finally {
  TransactionSynchronizationManager.clearSynchronization();
}
```

`initSynchronization()`으로 트랜잭션 동기화를 활성화한 뒤 서비스를 호출합니다. 이 시점에는 기존 이미지가 남아 있어야 하므로 `never()`로 삭제되지 않았음을 확인했습니다. 등록된 `afterCommit()`을 실행한 뒤에는 기존 이미지 삭제 호출을 검증했습니다.

롤백 테스트에서는 `afterCompletion(STATUS_ROLLED_BACK)`을 호출해 새 이미지가 삭제되는지 확인했습니다. `flush()`에서 예외가 발생하는 경우도 별도 테스트로 작성해 콜백 등록 전의 보상 작업까지 검사했습니다.

이 변경은 [`871a0648` 커밋](https://github.com/woowacourse-teams/2026-DAYNOMY/commit/871a0648d9e2b75832640ffbfabff59b69f7b1af)에 담겨 있습니다. 구현 코드 31줄과 테스트 코드 70줄을 추가하고 수정했습니다. 테스트 코드가 더 길어진 이유는 결과와 함께 커밋 전, 커밋 후, 롤백 후라는 세 시점을 각각 검증했기 때문입니다.

## 해결한 범위와 아직 남은 부분

`TransactionSynchronization`을 적용하면서 DB가 가리키는 이미지를 먼저 보호할 수 있게 되었습니다. 커밋되면 기존 이미지를 정리하고, 롤백되면 새 이미지를 정리합니다. 다만 DB와 S3가 원자성, 일관성, 고립성, 지속성(ACID)을 공유하는 하나의 트랜잭션이 된 것은 아닙니다. DB 처리 결과를 보고 S3 작업의 시점을 맞춘 구조입니다.

DB 커밋 뒤 `afterCommit()`에서 기존 이미지 삭제가 실패해도 DB를 되돌릴 수 없습니다. DAYNOMY는 삭제 예외를 잡아 경고 로그를 남깁니다. 사용자는 새 이미지를 계속 볼 수 있지만 사용하지 않는 기존 파일이 S3에 남을 수 있습니다. 이번 구현에서는 화면에 깨진 이미지를 노출할 위험을 줄이는 쪽에 우선순위를 두었습니다.

현재 구현에서 더 살펴봐야 할 상황도 있습니다.

- S3 업로드 직후 프로세스가 비정상 종료되면 `catch`와 콜백이 실행되지 않아 새 파일이 남을 수 있습니다.
- 이미지 삭제가 실패하면 경고 로그만 남고 자동으로 재시도하지 않습니다.
- `afterCompletion()`의 상태가 `STATUS_UNKNOWN`이면 새 이미지를 정리하지 않습니다.
- 콜백은 같은 요청 흐름에서 실행되므로 S3 응답이 느리면 요청 완료도 늦어집니다.

삭제 재시도와 복구까지 보장해야 한다면 메모리 안의 콜백만으로는 부족합니다. DB에 후속 작업을 함께 기록해 다시 처리하는 Outbox 패턴, 재시도 큐, 주기적인 고아 파일 정리 작업을 검토할 수 있습니다. S3 버전 관리를 켜서 잘못된 삭제에 대비할 수도 있지만, 참조되지 않는 파일을 정리하거나 애플리케이션의 일관성을 맞추는 일은 별도로 다뤄야 합니다.

이번 작업을 하면서 외부 시스템을 함께 변경할 때는 성공 순서만큼 실패 시점을 나누는 일이 중요하다는 것을 배웠습니다. 부작용이 확정되는 시점은 언제인지, 실패를 누가 감지하는지, 보상 작업이 실행되지 못하는 조건은 무엇인지 표로 적어 보니 구현과 테스트의 경계도 자연스럽게 정리됐습니다.

비슷한 문제를 다루고 있다면 정상 흐름 옆에 커밋 실패, 롤백, 외부 API 실패, 프로세스 종료 경로를 함께 그려보시길 권합니다. 빈칸으로 남은 경로가 있다면 그곳부터 다음 개선을 시작할 수 있습니다.

## 참고 자료

- [Spring Framework API: `TransactionSynchronization`](https://docs.spring.io/spring-framework/docs/current/javadoc-api/org/springframework/transaction/support/TransactionSynchronization.html)
- [Jakarta Persistence 3.2 명세](https://jakarta.ee/specifications/persistence/3.2/jakarta-persistence-spec-3.2.pdf)
- [AWS API Reference: `DeleteObject`](https://docs.aws.amazon.com/AmazonS3/latest/API/API_DeleteObject.html)
- [DAYNOMY 커밋: 뉴스 이미지 삭제 순서 조정](https://github.com/woowacourse-teams/2026-DAYNOMY/commit/871a0648d9e2b75832640ffbfabff59b69f7b1af)
