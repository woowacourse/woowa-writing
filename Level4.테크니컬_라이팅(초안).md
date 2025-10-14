# Playwright 기반 KBO 스코어보드 크롤러 성능 개선기

안녕하세요, 야구보구 팀의 밍트입니다.

야구보구는 사용자의 직관 기록을 분석해 개인화된 통계를 제공하는 서비스입니다.
매일 KBO 공식 웹사이트에서 실시간으로 경기 데이터를 수집해 최신 정보를 유지합니다.

하지만 초기 버전에서는 사용자가 체감하는 응답 속도가 만족스럽지 못했습니다. API 응답 시간이 평균 3.9초에 달했고, 특히 크롤링 구간에서 대부분의 시간이 소요되고 있었습니다. 이 글에서는 어떻게 응답 시간을 **51% 단축**시켜 1.9초로 개선했는지 과정을 공유하고자 합니다.

## 문제 인식: 왜 느렸을까?

먼저 병목 구간을 파악하기 위해 각 단계별 수행 시간을 측정했습니다.

```
StopWatch 'scoreboard:2025-10-04': 3.9초
- crawl: 3.5초 (90%)
- doubleHeaderOrder: 0.01초 (0%)
- upsertAll: 0.39초 (10%)
```

전체 시간의 90%가 크롤링에서 발생하고 있었습니다. 데이터베이스 저장은 이미 최적화되어 있었기 때문에, 개선의 방향은 명확했습니다. **Playwright 브라우저 자동화 구간을 최적화하자.**

크롤링 구간을 더 세밀하게 들여다봤습니다.

### 1. 매 요청마다 브라우저를 생성하고 종료

```java
public List<KboScoreboardGame> crawl(LocalDate date) {
    Playwright playwright = Playwright.create();  // 매번 생성
    Browser browser = playwright.chromium().launch();
    Page page = browser.newPage();
    
    // ... 크롤링 로직
    
    browser.close();   // 매번 종료
    playwright.close();
}
```

브라우저 인스턴스를 생성하는 데만 수백 밀리초가 소요됩니다. 하루에 수백 번 호출될 수 있는 API에서 이 오버헤드는 치명적이었습니다.

### 2. 불필요한 리소스 다운로드

페이지 로딩 시 이미지, 폰트, 동영상 같은 리소스를 모두 다운로드하고 있었습니다. 하지만 우리가 필요한 건 HTML과 일부 CSS뿐입니다. 수 MB에 달하는 불필요한 리소스가 네트워크 병목을 만들고 있었습니다.

### 3. 과도한 타임아웃

```java
page.setDefaultTimeout(30_000);  // 30초
page.setDefaultNavigationTimeout(60_000);  // 60초
```

경기가 없는 날짜에서도 긴 타임아웃을 기다리다가 실패하고 있었습니다. 실제로는 2~3초 안에 페이지 로딩이 완료되거나 실패하는데, 불필요하게 긴 대기 시간을 설정한 것이 문제였습니다.

## 해결 과정

### 1단계: 브라우저 컨텍스트 풀(Pool) 도입

첫 번째 개선은 **브라우저 인스턴스 재사용**입니다.

매 요청마다 브라우저를 생성하는 대신, 애플리케이션 시작 시 브라우저를 하나 띄워두고 여러 요청이 이를 공유하도록 변경했습니다. Playwright의 `BrowserContext`는 격리된 세션을 제공하므로, 여러 요청이 동시에 들어와도 쿠키나 캐시가 섞이지 않습니다.

```java
@Component
public class PlaywrightManager {
    private final Playwright pw;
    private final Browser browser;
    private final BlockingQueue<BrowserContext> contexts;

    public PlaywrightManager(@Value("${crawler.pool-size:3}") int poolSize) {
        this.pw = Playwright.create();
        this.browser = pw.chromium().launch(new BrowserType.LaunchOptions()
                .setHeadless(true)
                .setArgs(List.of(
                        "--disable-gpu",
                        "--disable-extensions",
                        "--no-sandbox",
                        "--disable-setuid-sandbox"
                ))
        );
        
        this.contexts = new ArrayBlockingQueue<>(poolSize);
        for (int i = 0; i < poolSize; i++) {
            BrowserContext ctx = browser.newContext(new Browser.NewContextOptions()
                    .setViewportSize(1280, 800)
                    .setUserAgent("Mozilla/5.0 ...")
                    .setBypassCSP(true));
            contexts.add(ctx);
        }
    }

    public Page acquirePage() throws InterruptedException {
        BrowserContext ctx = contexts.take();  // 풀에서 빌려옴
        return ctx.newPage();
    }

    public void releasePage(Page page) {
        if (page == null) return;
        BrowserContext ctx = page.context();
        try {
            page.close();  // 페이지만 닫음
        } catch (Exception ignore) {}
        contexts.offer(ctx);  // 컨텍스트는 풀에 반환
    }

    @PreDestroy
    public void shutdown() {
        try { browser.close(); } catch (Exception ignore) {}
        try { pw.close(); } catch (Exception ignore) {}
    }
}
```

`PlaywrightManager`를 Spring 빈으로 등록하고, 크롤러에서는 `acquirePage()`로 페이지를 빌려쓰고 `releasePage()`로 반환하도록 변경했습니다.

**효과**: 브라우저 콜드스타트 시간 완전 제거 → 약 500~800ms 단축

### 2단계: 불필요한 리소스 차단

두 번째 개선은 **네트워크 최적화**입니다.

Playwright의 `page.route()` API를 사용하면 특정 타입의 리소스 요청을 차단할 수 있습니다. 스코어보드 데이터 추출에는 HTML과 일부 JavaScript만 필요하므로, 이미지, 폰트, 미디어 파일을 모두 차단했습니다.

```java
public Page acquirePage() throws InterruptedException {
    BrowserContext ctx = contexts.take();
    Page page = ctx.newPage();

    // 불필요한 리소스 차단
    page.route("**/*", route -> {
        String type = route.request().resourceType();
        if ("image".equals(type) || 
            "media".equals(type) || 
            "font".equals(type)) {
            route.abort();  // 요청 중단
            return;
        }
        route.continue_();  // 필요한 요청만 계속 진행
    });

    page.setDefaultTimeout(6_000);
    page.setDefaultNavigationTimeout(10_000);
    return page;
}
```

**효과**: 페이지 로딩 시간 20~40% 단축 → 약 400~600ms 단축

### 3단계: 타임아웃 최적화와 빠른 스킵

세 번째 개선은 **불필요한 대기 제거**입니다.

경기가 없는 날짜에서는 스코어보드 요소가 존재하지 않습니다. 기존에는 긴 타임아웃을 기다리다가 예외가 발생했지만, 이제는 짧은 타임아웃으로 빠르게 확인하고 없으면 즉시 빈 결과를 반환합니다.

```java
private boolean waitForScoreboardsOrSkip(Page page, Duration timeout) {
    try {
        page.waitForSelector(".smsScore",
                new Page.WaitForSelectorOptions()
                        .setTimeout(timeout.toMillis())
                        .setState(WaitForSelectorState.ATTACHED));
        return true;
    } catch (PlaywrightException e) {
        return false;  // 타임아웃 발생 시 false 반환
    }
}

private List<KboScoreboardGame> extractScoreboards(Page page, Logger log, LocalDate date) {
    if (!waitForScoreboardsOrSkip(page, waitTimeout)) {
        return List.of();  // 데이터 없으면 즉시 반환
    }
    
    List<ElementHandle> scoreboards = page.querySelectorAll(".smsScore");
    // ... 파싱 로직
}
```

타임아웃도 실제 필요한 수준으로 줄였습니다.

```java
page.setDefaultTimeout(6_000);              // 6초
page.setDefaultNavigationTimeout(10_000);   // 10초
```

**효과**: 빈 날짜 처리 시간 대폭 단축, 평균 응답 시간 안정화

### 4단계: 캘린더 네비게이션 안정화

KBO 웹사이트는 날짜 선택에 jQuery UI Datepicker를 사용합니다. 기존 코드는 CSS 선택자로 날짜를 찾았는데, 달력에 이전 달/다음 달의 날짜가 함께 표시될 때 잘못된 날짜를 클릭하는 버그가 있었습니다.

```java
// 기존: CSS 선택자 (문제 있음)
page.click("a:text('" + day + "')");

// 개선: XPath로 현재 달의 날짜만 선택
String dayXpath = String.format(
    "//div[@id='ui-datepicker-div']" +
    "//td[not(contains(@class,'ui-datepicker-other-month'))]" +
    "//a[normalize-space(text())='%s']",
    day
);
page.click(dayXpath);
```

XPath를 사용해 `ui-datepicker-other-month` 클래스가 없는(현재 달인) 셀만 선택하도록 변경했습니다.

날짜 선택 후에는 페이지가 제대로 로딩되었는지 확인하는 검증 로직도 추가했습니다.

```java
// 날짜 라벨이 제대로 표시되었는지 확인
String expected = "2025.10.04";  // 기대하는 날짜 형식
page.waitForFunction(
    "(args) => {" +
    "  const [exp, sel] = args;" +
    "  const el = document.querySelector(sel);" +
    "  return el && el.textContent && el.textContent.includes(exp);" +
    "}",
    new Object[]{expected, "#cphContents_cphContents_cphContents_lblGameDate"},
    new Page.WaitForFunctionOptions().setTimeout(waitTimeout.toMillis())
);
```

**효과**: 간헐적으로 발생하던 날짜 선택 실패 완전 제거

## 최종 결과

모든 개선을 적용한 후의 성능 측정 결과입니다.

### API 전체 응답 시간

```
StopWatch 'scoreboard:2025-10-04': 1.855초
------------------------------------------------------
Seconds       %       Task name
------------------------------------------------------
1.719초       93%     crawl
0.001초       00%     doubleHeaderOrder
0.136초       07%     upsertAll
```

**3.9초 → 1.9초 (51% 개선)**

### 크롤링 단계별 시간

```
StopWatch 'crawl:2025-10-04': 1.713초
-------------------------------------------------
Seconds       %       Task name
-------------------------------------------------
1.216초       71%     navigate
0.313초       18%     calendarNav
0.184초       11%     extract
```

크롤링 구간도 **3.5초 → 1.7초**로 개선되었습니다. 페이지 네비게이션이 여전히 71%를 차지하지만, 브라우저 재사용과 리소스 차단으로 절대 시간을 크게 줄였습니다.

## 개선 효과 정리

|항목|개선 전|개선 후|감소율|
|---|---|---|---|
|API 응답 시간|3.9초|1.9초|51%|
|크롤링 시간|3.5초|1.7초|51%|
|네비게이션|~2.5초|1.2초|52%|

사용자 체감 속도가 절반 이하로 줄어들었고, 서버 리소스 사용도 크게 감소했습니다.

## 구현 세부사항

### 재시도 메커니즘

네트워크 불안정이나 일시적인 오류에 대응하기 위해 재시도 로직을 구현했습니다.

```java
public List<KboScoreboardGame> crawlScoreboard(LocalDate date, Logger logger) {
    StopWatch sw = new StopWatch("crawl:" + date);
    
    for (int attempt = 1; attempt <= maxRetries; attempt++) {
        Page page = null;
        try {
            sw.start("navigate");
            page = pwManager.acquirePage();
            page.navigate(BASE_URL, 
                new Page.NavigateOptions().setTimeout(navigationTimeout.toMillis()));
            sw.stop();

            sw.start("calendarNav");
            navigateToDateUsingCalendar(page, date, waitTimeout);
            sw.stop();

            sw.start("extract");
            List<KboScoreboardGame> games = extractScoreboards(page, logger, date);
            sw.stop();

            logger.info("[UPSERT] date={} phases={} total={}ms size={}", 
                date, sw.prettyPrint(), sw.getTotalTimeMillis(), games.size());
            
            return games;
            
        } catch (PlaywrightException e) {
            logger.warn("크롤링 오류 발생(시도 {}/{}): {}", 
                attempt, maxRetries, e.getMessage());
            
            if (attempt == maxRetries) {
                logger.info("[CRAWL_EMPTY] date={} 데이터 없음", date);
                break;
            }
            sleep(retryDelay);
            
        } finally {
            if (page != null) {
                pwManager.releasePage(page);
            }
        }
    }
    
    return List.of();
}
```

최대 3회까지 재시도하며, 각 시도 사이에는 1초의 지연을 둡니다. `finally` 블록에서 페이지를 반드시 반환해 리소스 누수를 방지합니다.

### 방어적인 데이터 파싱

웹 크롤링은 언제든 HTML 구조가 변경될 수 있기 때문에 방어적으로 코드를 작성했습니다.

```java
private String safeText(ElementHandle parent, String selector) {
    if (parent == null) return null;
    try {
        ElementHandle element = parent.querySelector(selector);
        if (element == null) return null;
        
        String text = element.innerText();
        return text != null ? text.trim() : null;
    } catch (PlaywrightException e) {
        return null;
    }
}

private Integer parseNullableInt(String text) {
    if (text == null) return null;
    
    String normalized = text.replaceAll("[^0-9-]", "").trim();
    if (normalized.isEmpty() || "-".equals(normalized)) {
        return null;
    }
    
    try {
        return Integer.parseInt(normalized);
    } catch (NumberFormatException e) {
        return null;
    }
}
```

모든 요소 접근과 파싱에서 null을 허용하고, 예외가 발생해도 크롤링 전체가 실패하지 않도록 처리했습니다.

## 추가 개선 가능 영역

### 병렬 처리

현재는 한 번에 하나의 날짜만 크롤링합니다. 여러 날짜를 동시에 조회해야 하는 경우, 컨텍스트 풀 크기를 늘리고 병렬 처리를 적용할 수 있습니다.

```java
public List<KboScoreboardGame> crawlMultipleDates(List<LocalDate> dates) {
    return dates.parallelStream()
        .flatMap(date -> crawlScoreboard(date).stream())
        .collect(Collectors.toList());
}
```

다만 KBO 웹사이트에 과도한 부하를 주지 않도록 주의해야 합니다.

### 캐싱 전략

확정된 과거 경기는 변경되지 않습니다. Redis나 로컬 캐시를 도입하면 반복 요청에서 크롤링을 건너뛸 수 있습니다.

```java
@Cacheable(value = "scoreboards", key = "#date")
public List<KboScoreboardGame> crawlScoreboard(LocalDate date) {
    // ...
}
```

### CDP(Chrome DevTools Protocol) 활용

Playwright는 내부적으로 CDP를 사용합니다. 더 세밀한 네트워크 제어가 필요하다면 CDP를 직접 사용하는 것도 고려할 수 있습니다.

## 마치며

이번 개선을 통해 **API 응답 시간을 51% 단축**할 수 있었습니다. 핵심은 다음 네 가지였습니다.

1. **브라우저 재사용**: 콜드스타트 제거
2. **리소스 차단**: 불필요한 네트워크 트래픽 제거
3. **타임아웃 최적화**: 빠른 실패와 빠른 응답
4. **안정적인 네비게이션**: XPath 기반 정확한 요소 선택

Playwright 같은 브라우저 자동화 도구는 강력하지만 느릴 수 있습니다. 이번 사례가 유사한 크롤링 시스템을 개발하거나 최적화하는 분들에게 도움이 되었으면 좋겠습니다.

긴 글 읽어주셔서 감사합니다.
