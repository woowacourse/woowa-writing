# **웹 폰트 로딩 성능 최적화 전략**

## 0) 목표

웹 폰트(Web Font)의 로딩 성능과 렌더링 안정성을 고려한 최적화 전략을 직접 설계하고 적용할 수 있는 수준에 도달한다.

---

## 1) 실습 환경 준비

### 1.1 폴더 구조

```
font-optimization/
 ┣ index.html
 ┣ style.css
 ┗ fonts/
```

### 1.2 폰트 파일 준비

#### 예시 1: Pretendard

* **다운로드:** [https://github.com/orioncactus/pretendard/releases](https://github.com/orioncactus/pretendard/releases)
* “PretendardSubset”의 **WOFF2(Web Open Font Format 2)** 버전 다운로드
* `Pretendard-Regular.woff2` 파일을 `fonts/` 폴더에 저장

#### 예시 2: Noto Sans KR (Google Fonts)

* **링크:** [https://fonts.google.com/noto/specimen/Noto+Sans+KR](https://fonts.google.com/noto/specimen/Noto+Sans+KR)
* 압축을 해제한 후 `NotoSansKR-Regular.woff2` 파일을 `fonts/` 폴더에 저장

최종 구조:

```
font-optimization/
 ┣ index.html
 ┣ style.css
 ┗ fonts/
    ┣ Pretendard-Regular.woff2
    ┗ NotoSansKR-Regular.woff2
```

---

### 1.3 HTML 기본 구성

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>폰트 최적화 실습</title>
    <link rel="stylesheet" href="style.css" />
  </head>
  <body>
    <h1>폰트 최적화 실습 🎨</h1>
    <p>
      이 문장은 Pretendard 폰트로 렌더링됩니다. 
      폰트 로딩 과정을 직접 관찰합니다.
    </p>
  </body>
</html>
```

---

### 1.4 CSS 연결 확인

```css
body {
  background: #fafafa;
  color: #222;
  font-family: sans-serif;
}
```

> Visual Studio Code에서 **“Open with Live Server”** 로 실행 후 페이지를 확인한다.

---

## 2) 폰트 포맷 비교

| 포맷                              | 특징                    | 장점                  | 주의점                  |
| ------------------------------- | --------------------- | ------------------- | -------------------- |
| **TTF (TrueType Font)**         | 데스크탑용 기본 폰트 포맷        | 높은 호환성              | 용량 큼, 웹 비최적화         |
| **OTF (OpenType Font)**         | TTF 기반 + 고급 타이포그래피 기능 | 다양한 글자 조합, 인쇄 품질 우수 | 웹 성능 비효율             |
| **WOFF (Web Open Font Format)** | 웹 전용 압축 포맷            | 압축 + 호환성            | WOFF2보다 비효율적         |
| **WOFF2**                       | WOFF 개선판, 최신 브라우저 지원  | 압축률 높음, 웹 표준        | 구형 브라우저(IE11 이하) 미지원 |

> **결론:** 웹에서는 **WOFF2** 포맷이 기본 표준이다.
> TTF 또는 OTF는 직접 사용하지 말고 변환 후 사용하는 것이 권장된다.

---

## 3) 웹 폰트 로드 방식 이해

웹 폰트를 로드하는 대표적인 방식은 두 가지이다.

### A. 로컬 폰트 파일 직접 사용

```css
@font-face {
  font-family: 'Pretendard';
  src: url('./fonts/Pretendard-Regular.woff2') format('woff2');
  font-display: swap;
}
```

* `font-family`: 웹에서 사용할 이름 정의
* `src`: 실제 폰트 파일 경로 및 형식 지정
* `format('woff2')`: 브라우저에 파일 형식 명시
* `font-display: swap`: 로딩 중 시스템 폰트를 표시 후 교체

**장점**

* 파일 직접 제어 가능 (서브셋, 캐싱 전략 적용)
* 외부 네트워크 의존 없음

**단점**

* CDN 캐싱 효과 없음
* 서버 부하 및 관리 책임 발생

---

### B. 외부 CDN 사용

```html
<link
  rel="stylesheet"
  href="https://cdn.jsdelivr.net/gh/orioncactus/pretendard@v1.3.9/dist/web/static/pretendard.min.css"
/>
```

**장점**

* 자동 압축 및 전역 캐싱
* 빠른 적용 가능

**단점**

* 외부 의존성 존재
* 폰트 수정 및 서브셋 작업 어려움

---

### C. 결론

| 구분         | 장점                   | 단점       | 권장 상황          |
| ---------- | -------------------- | -------- | -------------- |
| **로컬 관리**  | 커스터마이징 용이, 성능 최적화 가능 | 직접 관리 필요 | 서비스 품질에 민감한 경우 |
| **CDN 사용** | 손쉬운 적용, 유지보수 편리      | 외부 의존성   | 초기 구축 또는 프로토타입 |

---

## 4) 폰트 로딩 구조 및 `font-display` 동작

브라우저는 텍스트 렌더링 시 폰트 로딩 상태에 따라 **FOIT(Flash of Invisible Text)** 또는 **FOUT(Flash of Unstyled Text)** 현상을 보인다.

| 값        | 동작 방식               | 특징             |
| -------- | ------------------- | -------------- |
| auto     | 브라우저 기본 동작          | 브라우저마다 다름      |
| block    | 로드 전 텍스트 숨김 → 이후 교체 | FOIT 발생 가능     |
| swap     | 시스템 폰트 표시 후 교체      | UX 향상, 현대적 기본값 |
| fallback | 짧은 block 후 swap     | 대부분 상황에서 권장    |
| optional | 느리면 시스템 폰트 유지       | 저속 네트워크에 적합    |

> 💡 실무에서는 `font-display: swap;` 속성이 가장 널리 사용된다.

---

## 5) DevTools를 통한 로딩 타이밍 분석

1. Chrome DevTools → **Network 탭 → Filter “Fonts”**
2. 페이지 새로고침 후 `Pretendard-Regular.woff2` 요청 시점 관찰
3. **Throttling**을 “3G”로 설정하여 느린 네트워크 환경 시뮬레이션
4. `font-display`별 렌더링 차이 비교

| 설정       | 관찰 결과                        |
| -------- | ---------------------------- |
| swap     | 시스템 폰트로 즉시 표시 후 교체 (FOUT 발생) |
| block    | 로딩 전 텍스트 미표시 (FOIT 발생)       |
| fallback | 짧은 숨김 후 교체                   |
| optional | 폰트 느릴 시 교체 생략                |

---

## 6) 폰트 포맷 변환 및 용량 최적화

### 6.1 변환 방법

#### (A) 온라인 변환 — Transfonter.org

1. [https://transfonter.org](https://transfonter.org) 접속
2. `.ttf` 업로드 후 출력 포맷으로 **WOFF2** 선택
3. 변환 후 `fonts/` 폴더에 저장

#### (B) CLI 변환 — macOS

```bash
brew install woff2
woff2_compress NotoSansKR-Regular.ttf
```

결과: `NotoSansKR-Regular.woff2` 생성

---

### 6.2 용량 비교 예시

| 포맷    | 용량    |
| ----- | ----- |
| TTF   | 6.2MB |
| WOFF2 | 2.1MB |

> ➜ 약 **3배 압축률 확보**, 로딩 및 렌더링 시간 단축

---

## 7) 폰트 서브셋(Subsetting)

### 7.1 개념

전체 폰트에서 실제 사용하는 글자만 포함하여 **용량을 줄이는 기법**.

### 7.2 한글 조합 수

* 초성(19) × 중성(21) × 종성(28) = **11,172자**
* 불필요한 문자를 제거하면 최대 **60~70% 용량 절감 가능**

### 7.3 서브셋 도구

* [https://t.hi098123.com/font-subset](https://t.hi098123.com/font-subset)
* 선택한 글자만 포함하는 **WOFF2 파일 생성 가능**

### 7.4 사례

| 구분     | 파일                       | 용량    |
| ------ | ------------------------ | ----- |
| 전체 폰트  | Pretendard-Regular.woff2 | 766KB |
| 서브셋 폰트 | Pretendard-Subset.woff2  | 267KB |

> ➜ 약 **3배 감소 (LCP 및 초기 렌더링 개선)**

---

## 8) Preload를 통한 로딩 우선순위 제어

### 8.1 기본 방식

```html
<link href="./fonts/Pretendard-Regular.woff2" rel="stylesheet">
```

> CSS 파싱 후 로드되므로 폰트 다운로드가 지연될 수 있다.

### 8.2 Preload 적용

```html
<link
  rel="preload"
  href="./fonts/Pretendard-Regular.woff2"
  as="font"
  type="font/woff2"
  crossorigin="anonymous"
/>
```

**장점**

* HTML 파싱 중 폰트 다운로드 시작
* 렌더링 차단 최소화
* 캐시 활용으로 중복 다운로드 없음

| 구분        | preload 사용   | preload 미사용 |
| --------- | ------------ | ----------- |
| 다운로드 시점   | HTML 파싱 중 즉시 | CSS 해석 후    |
| 렌더링 차단    | 없음           | 일부 지연       |
| 초기 텍스트 표시 | 빠름           | 늦음          |

---

## 9) 불필요한 폰트 굵기 제거

Pretendard CSS에는 **100~900**까지 모든 `font-weight`가 정의되어 있다.
프로젝트에서 실제 사용하는 굵기(예: 400, 500, 600, 700)만 남기면 리소스 낭비를 줄일 수 있다.

```css
/* Regular 400 */
@font-face {
  font-family: 'Pretendard';
  font-weight: 400;
  font-display: swap;
  src: url('./fonts/Pretendard-Regular.woff2') format('woff2');
}

/* Medium 500 */
@font-face {
  font-family: 'Pretendard';
  font-weight: 500;
  font-display: swap;
  src: url('./fonts/Pretendard-Medium.woff2') format('woff2');
}

/* SemiBold 600 */
@font-face {
  font-family: 'Pretendard';
  font-weight: 600;
  font-display: swap;
  src: url('./fonts/Pretendard-SemiBold.woff2') format('woff2');
}

/* Bold 700 */
@font-face {
  font-family: 'Pretendard';
  font-weight: 700;
  font-display: swap;
  src: url('./fonts/Pretendard-Bold.woff2') format('woff2');
}
```

> `preload`로 다운로드된 파일은 브라우저 캐시에 저장되므로, CSS에서 같은 파일을 다시 요청하더라도 **중복 다운로드는 발생하지 않는다.**

---

## 10) 구형 브라우저 호환 설정

```css
@font-face {
  font-family: 'Pretendard';
  font-weight: 400;
  font-display: swap;
  src: 
    url('./Pretendard-Regular.woff2') format('woff2'),
    url('./Pretendard-Regular.woff') format('woff'),
    url('./Pretendard-Regular.ttf') format('truetype');
}
```

브라우저는 상단부터 순차적으로 지원 여부를 검사하여 가능한 첫 번째 포맷을 선택한다.

* 최신 브라우저: **WOFF2 사용**
* 구형 브라우저: **WOFF 또는 TTF로 폴백(fallback)**