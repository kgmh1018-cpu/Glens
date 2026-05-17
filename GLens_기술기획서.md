# GLens — Chrome 확장 프로그램 기술 기획서 (초안)

---

## 1. 프로젝트 개요

웹페이지 이미지에서 Apple Live Text처럼 텍스트를 바로 선택·복사할 수 있는 Chrome 확장 프로그램. Chrome + Windows 환경 타겟. 개인 사용 및 지인 배포 목적.

---

## 2. 기존 북마클릿과의 차이

| 항목 | 북마클릿 | Chrome 확장 |
|------|---------|------------|
| 핫링크 차단 이미지 | Google Lens URL 전달 → 실패 | background.js에서 `<all_urls>` 호스트 권한으로 직접 fetch → 해결 |
| OCR | Google Lens 새탭 | Google Cloud Vision API → 모달 내 처리 |
| 텍스트 선택 | 불가 | 드래그 선택 + 실시간 선택 텍스트 패널 |
| 외부 이미지 | 불가 | 파일 드래그 앤 드롭 / 클릭 업로드 지원 |
| 단축키 | 북마클릿 클릭 | 네이티브 단축키 |

---

## 3. 파일 구조

```
manifest.json   — 확장 설정, 단축키, content script 로드 순서, 호스트 권한
background.js   — 이미지 fetch (CORS/핫링크 우회), Vision API 호출
content.js      — 진입/종료, 이벤트 총괄
modal.js        — 모달 UI 생성/제어
overlay.js      — 텍스트 오버레이 레이어
panel.js        — 선택 텍스트 실시간 표시 패널
content.css     — 모달/오버레이/패널 스타일, 커서 상태, ::selection 스타일
options.html    — 설정 페이지 UI
options.js      — 설정 저장/로드
```

### content script 로드 순서 (manifest.json에 명시)

```json
"content_scripts": [{
  "css": ["content.css"],
  "js": ["overlay.js", "panel.js", "modal.js", "content.js"]
}]
```

overlay.js → panel.js → modal.js → content.js 순서로 로드. content.js가 나머지 파일의 함수를 사용하므로 반드시 마지막에 로드.

### 호스트 권한 (manifest.json에 명시)

```json
"host_permissions": ["<all_urls>"]
```

background.js에서 임의 도메인의 이미지를 fetch하기 위해 필요. 이 권한이 없으면 cross-origin fetch가 브라우저 보안 정책에 의해 차단됨. 크롬 웹스토어 심사에서 민감하게 보는 권한이므로 배포 시 유의.

---

## 4. 전역 네임스페이스 구조

모든 파일은 `window._glens` 하나로 통신. 확장 내부 파일 간 통신 구조화 및 추후 기능 추가 시 기존 코드 수정 없이 확장 가능하도록 설계.

참고: Content Script는 크롬 확장의 구조상 원본 페이지 스크립트와 완전히 분리된 격리된 환경(Isolated World)에서 실행되므로, 페이지 내 기존 스크립트와 `window` 변수가 충돌하는 일은 구조적으로 발생하지 않음.

```javascript
window._glens = {
  // content.js 관할
  active: false,
  currentRequest: null,

  enter() {},
  exit() {},    // 호출 시 modal DOM 제거, canvas 객체 초기화,
                // 대용량 base64 변수 null 할당으로 GC 회수 유도

  // modal.js 관할
  modal: {
    open(imageData) {},
    close() {},
    showLoading() {},
    showError(message) {},
    showNoText() {},
  },

  // overlay.js 관할
  overlay: {
    render(words, imageEl) {},
    clear() {},
  },

  // panel.js 관할
  panel: {
    show() {},
    hide() {},
    update(text) {},   // selectionchange 이벤트로 실시간 갱신
  },
}
```

추후 기능 추가 시 이 객체에 네임스페이스만 추가하면 됨. 예: `window._glens.history = {}`, `window._glens.translate = {}`

---

## 5. 설정 구조

모든 설정값은 `chrome.storage.sync`로 관리. 크롬 계정 동기화 지원.

```javascript
{
  apiKey: '',
  escBehavior: 'fullExit',  // 'fullExit' | 'modalOnly'
}
```

추후 설정 추가 시 이 객체에만 항목 추가. 설정 페이지(options.html)에서 렌더링.

**설정 로딩 타이밍:** content.js 초기 실행 시 chrome.storage.sync에서 설정값 즉시 로드 및 캐싱. 로드 완료 전 단축키 입력 시 기본값으로 동작.

---

## 6. UX 흐름

```
1. 단축키 입력
   → API 키 존재 여부 확인
   → 없으면: 모드 진입 차단 + 토스트 "🔑 API 키를 먼저 설정해주세요" + options 페이지 자동 오픈
   → 있으면: 이미지 인식 모드 진입

2. 이미지 인식 모드 진입
   → document.body.classList.add('_glens-active') 로 클래스 토글
     (content.css에서 ._glens-active * { cursor: zoom-in !important } 정의)
   → 하단에 인터랙티브 토스트 표시 (이미지 클릭 또는 파일 선택 전까지 유지)

     ┌─────────────────────────────────────────┐
     │  🔍 이미지를 클릭하세요  ·  ESC로 취소   │
     │  ─────────────────────────────────────  │
     │     📎 파일 드래그 또는 클릭해서 선택    │
     └─────────────────────────────────────────┘

     - 파일 드래그 오버 시 토스트 테두리 파란색으로 변경
     - 이미지 클릭 또는 파일 선택 시 토스트 닫힘 + 모달 오픈
   → 크롬 확장 아이콘 배지 "ON" 표시
   → body scroll lock 적용

3. 이미지 호버
   → 가로 × 세로 >= 10,000px² 이상인 이미지에만 파란 테두리 표시
   → 오버레이 mousemove + pointer-events 토글 + elementFromPoint 방식

4. 이미지 클릭 또는 파일 선택
   → 모달 즉시 오픈
   → 모달 안에 이미지 표시 + 로딩 스피너
   → 선택 텍스트 패널 표시 (모달 우측에 독립 DOM으로 렌더링)

5. OCR 처리
   → background.js에서 이미지 소스 취득 + Vision API 호출
   → 응답 도착 시:
      - 텍스트 있음: 이미지 위에 텍스트 오버레이 표시
      - 텍스트 없음: "텍스트를 찾을 수 없습니다" 문구 표시
      - 오류: "이미지를 불러올 수 없습니다" 또는 "OCR 처리 중 오류가 발생했습니다" 표시

6. 텍스트 선택
   → 드래그 선택 시 selectionchange 이벤트로 선택 텍스트 실시간 감지
   → 모달 우측 패널에 선택 텍스트 즉시 표시
   → 패널 내 복사 버튼 또는 Ctrl+C로 복사

7. 모달 닫기
   → 닫기 버튼 / ESC / 모달 바깥 클릭
   → 선택 텍스트 패널도 함께 닫힘
   → ESC 동작은 설정에서 변경 가능:
      - fullExit (기본값): 모달 + 인식 모드 전체 종료
      - modalOnly: 모달만 닫힘, 인식 모드 유지

8. 인식 모드 종료
   → document.body.classList.remove('_glens-active') 로 커서 복구
   → 배지 제거
   → body scroll lock 해제
```

---

## 7. 취소 방법

세 가지 모두 동일하게 `window._glens.exit()` 호출. 우선순위 없음.

- ESC (설정에 따라 모달만 or 전체 종료)
- 단축키 재입력
- 빈 곳 클릭 (mousedown까지 차단해서 페이지 무상호작용 취소)

---

## 8. 이미지 인식 모드 상태 관리

```javascript
window._glens.active         // true/false
window._glens.currentRequest // 현재 진행 중인 요청 추적용 변수
```

새 이미지 클릭 시 currentRequest를 새 값으로 덮어씀. background.js 응답 도착 시 currentRequest와 비교해서 다르면 무시 (로딩 중 ESC로 닫거나 다른 이미지 클릭한 경우 처리). 이 구조로 중복 API 호출도 방지됨.

탭 이동 시 인식 모드 유지. content script는 탭별로 독립 실행되므로 별도 처리 불필요.

**배지 탭별 독립 관리 필수:** `chrome.action.setBadgeText`에 반드시 `tabId` 명시. 미명시 시 전체 탭에 배지가 적용되어 모드가 꺼진 탭에서도 "ON"이 표시되는 버그 발생.

```javascript
chrome.action.setBadgeText({ text: 'ON', tabId: sender.tab.id })
chrome.action.setBadgeText({ text: '', tabId: sender.tab.id })
```

---

## 9. 이미지 소스 추출 및 처리 흐름

content.js가 클릭된 요소의 형태를 파악해 소스를 추출한 뒤 background.js로 전달.

```
content.js — 클릭된 요소 파악
  ├── img 태그         → currentSrc || src (URL)
  │                      (currentSrc 우선: srcset 반응형 이미지에서 브라우저가
  │                       실제 선택한 URL과 src가 다를 수 있음)
  ├── background-image → CSS 문자열에서 URL 파싱
  ├── canvas           → toDataURL() (base64 직접 추출)
  └── 파일 업로드      → FileReader로 base64 변환

URL 케이스    → URL만 background.js로 전달
               → Vision API에 URL 직접 전송 (1순위, 전처리 없음)
               → 실패 시: background.js가 <all_urls> 권한으로 fetch
                         → 이미지 처리 파이프라인 → base64 전송 (폴백)

base64 케이스 → background.js로 전달
(canvas/파일)  → 이미지 처리 파이프라인 → Vision API 호출

background.js → 텍스트 좌표 JSON → content.js
```

**URL 직접 전송이 1순위인 이유:** Vision API는 이미지 URL을 직접 수신해 Google 서버에서 fetch한다. 이미지 데이터를 content.js → background.js → Vision API로 이중 전송할 필요가 없어 가장 빠르다.

**URL 직접 전송의 한계:** 핫링크 차단 이미지는 Google 서버 IP도 차단되어 실패할 수 있다. 이 경우 background.js가 `<all_urls>` 권한으로 직접 fetch해 base64로 전환하는 폴백을 실행한다.

**원본 전체 이미지 처리:** 소스에서 직접 추출하므로 모니터 화면에 이미지가 반만 걸쳐 있거나 스크롤 아래에 있어도 원본 파일 전체 데이터를 100% 확보.

**background.js에서 OffscreenCanvas 사용:** background.js는 서비스 워커로 DOM이 없으므로 일반 canvas 불가. OffscreenCanvas API 사용 (Chrome 확장 서비스 워커에서 공식 지원).

### 이미지 처리 파이프라인 (base64 전송이 필요한 경우)

**Vision API 실제 제약:**
- JSON 요청 크기 한도: 10MB
- base64 인코딩 시 원본 대비 약 37% 크기 증가 → base64 전송 가능한 원본 이미지 실질 상한: 약 7.3MB
- OCR 분석 픽셀 상한: 7,500만 픽셀 (초과 시 API가 자동 리사이즈)
- 공식 권장 최소 해상도: 640×480px

OCR 정확도는 텍스트의 픽셀 너비에 의존하므로 품질(85%)을 유지한 채 해상도로만 크기를 조절한다.

```
Step 1. 너비 조정 (너비 > maxWidth인 경우에만)
        → maxWidth로 축소 (세로는 비율 유지)

Step 2. JPEG 85% 단일 인코딩
        → 결과 < maxPayloadBytes: 즉시 전송 (대부분 여기서 종료)

Step 3. 안전망 (Step 2 후에도 초과 시, 극단적 케이스)
        → 너비를 minWidth까지 단계 축소 후 85% 재인코딩 → 전송
        (품질은 85% 유지, 해상도로만 조절)
```

세로가 긴 이미지(웹툰 등)는 너비만 기준으로 처리하므로 별도 분기 불필요.

### 이미지 처리 상수 (background.js)

처리 파라미터를 상수 객체로 중앙화하여 추후 튜닝 시 한 곳만 수정.

```javascript
const IMAGE_PROCESSING = {
  quality:         0.85,              // JPEG 품질 (OCR 안전 하한 고려, 고정)
  maxWidth:        1600,              // 너비 축소 상한
  minWidth:        800,               // 너비 축소 하한 (공식 권장 640px 이상 유지)
  maxPayloadBytes: 7 * 1024 * 1024,  // JSON 10MB 한도, base64 37% 오버헤드 감안
};
```

---

## 10. 모달 및 선택 텍스트 패널 구조

### 모달

모달 안에 이미지를 새로 렌더링하는 방식. 원본 페이지의 CSS(object-fit 등)와 완전히 분리되어 좌표 계산이 단순해짐.

**모달 표시 이미지와 Vision API 전송 이미지 통일:**
- 동일한 이미지를 모달 표시와 Vision API 전송에 모두 사용
- 좌표 계산 시 비율 변환 불필요, 픽셀 1:1 매핑

**로딩 상태:**
```
모달 열림 → 이미지 표시 + 로딩 스피너 → OCR 완료 → 텍스트 오버레이
```

**모달 스타일:**
- `prefers-color-scheme`으로 라이트/다크 모드 자동 대응
- CSS 클래스명 전부 `_glens-` 접두사로 통일 (페이지 CSS 충돌 방지)
- `._glens-modal` 루트에 CSS 초기화 명시 (font-size, line-height, box-sizing 등 기본값 고정)
- 모달 열릴 때 body 스크롤 잠금, 닫힐 때 복구
- 구현: 순수 CSS + JS (React 등 프레임워크 사용 안 함)

### 선택 텍스트 패널 (panel.js)

모달과 독립된 별도 DOM 요소. 모달 우측에 살짝 어긋나게 배치하여 이미지를 가리지 않으면서 선택 텍스트를 표시.

```
          ┌──────────────────────────┐
          │                          │  ┌─────────────┐
          │  이미지 + 텍스트 오버레이  │  │ 선택된 텍스트│
          │                          │  │             │
          │                          │  │   [복사] 📋 │
          └──────────────────────────┘  └─────────────┘
```

- `selectionchange` 이벤트로 선택 텍스트 실시간 감지 → `panel.update(text)` 호출
- 선택된 텍스트가 없을 때는 안내 문구 표시: "텍스트를 드래그해서 선택하세요"
- 복사 버튼: `navigator.clipboard.writeText()` 사용
- 모달 닫힐 때 패널도 함께 닫힘

---

## 11. 텍스트 오버레이 (overlay.js)

**참고 패턴:** PDF.js 텍스트 레이어 방식

**구현 방식:**
```
img.onload 완료 후 모달 이미지 실제 표시 크기 측정
→ 스케일 계산
→ 이미지 위에 투명 div 레이어 하나 (position: absolute)
→ 각 단어마다 absolute 포지션 span 생성
→ 단어 사이 공백도 투명 span으로 채워 드래그 선택이 끊기지 않게 처리
→ 브라우저 기본 드래그 선택 동작
```

**좌표 계산:**

모달에서 이미지가 CSS로 축소 표시될 수 있으므로 스케일 계산 필수.

```
scaleX = img.getBoundingClientRect().width  / 이미지 실제 픽셀 너비
scaleY = img.getBoundingClientRect().height / 이미지 실제 픽셀 높이

span.style.left   = (x1 * scaleX) + 'px'
span.style.top    = (y1 * scaleY) + 'px'
span.style.width  = ((x2 - x1) * scaleX) + 'px'
span.style.height = ((y2 - y1) * scaleY) + 'px'
```

**회전 텍스트 (초기 버전부터 구현):**

DOCUMENT_TEXT_DETECTION이 꼭짓점 4개를 이미 제공하므로 추가 비용 없이 초기부터 구현.

```
v = boundingPoly.vertices  // [좌상단, 우상단, 우하단, 좌하단]
각도 = Math.atan2(v[1].y - v[0].y, v[1].x - v[0].x)
span.style.transform = `rotate(${각도}rad)`
span.style.transformOrigin = 'top left'
```

**텍스트 선택 시각적 피드백:**

content.css에 `::selection` 스타일을 추가하여 드래그 시 선택 영역이 시각적으로 표시되도록 처리.

```css
._glens-overlay span::selection {
  background-color: rgba(66, 133, 244, 0.3);
}
```

**img.onload 타이밍 중요:** 이미지가 모달에 완전히 로드된 후에 좌표 계산 시작. 로드 전 계산 시 이미지 크기가 0이라 오버레이 전부 좌상단에 몰리는 버그 발생.

---

## 12. 메모리 관리

`window._glens.exit()` 호출 시 아래 항목을 명시적으로 처리하여 GC가 메모리를 회수할 수 있도록 함.

- 모달 내 DOM 요소 제거
- 선택 텍스트 패널 DOM 제거
- OffscreenCanvas 객체 초기화
- 대용량 base64 문자열을 담은 변수에 `null` 할당

Vision API 처리 과정에서 base64 변환 및 메시징으로 인해 메모리 점유율이 일시적으로 높아지므로, 모드 종료 시 명시적 해제가 필요. AI 코드 생성 시 빠뜨리기 쉬운 부분.

---

## 13. background.js ↔ content.js 메시지 구조

```javascript
// content.js → background.js (URL 케이스)
{ action: 'processImage', imageUrl: 'https://...', requestId: 'abc123' }

// content.js → background.js (base64 케이스: canvas, 파일 업로드)
{ action: 'processImage', imageData: 'base64...', requestId: 'abc123' }

// background.js → content.js (성공)
{ action: 'ocrResult', words: [...], imageData: 'base64...', requestId: 'abc123' }

// background.js → content.js (실패)
{ action: 'ocrError', error: 'fetch_failed' | 'ocr_failed', requestId: 'abc123' }
```

---

## 14. Vision API 연동

**사용 기능:** `DOCUMENT_TEXT_DETECTION` (문서, 스크린샷, 손글씨 등 실제 사용 케이스에 최적화. 단락→줄→단어→글자 계층 구조 및 꼭짓점 4개 제공)

**이미지 전송 방식:**
- 1순위: URL 문자열 직접 전송 (Google 서버가 fetch, 가장 빠름)
- 폴백: base64 인코딩 전송 (핫링크 차단 이미지 등 URL 전송 실패 시)

**응답 구조 활용:**
```javascript
response.textAnnotations        // 전체 텍스트
response.textAnnotations[1..n]  // 단어별 (boundingPoly.vertices 포함)
```

**API 키:** 사용자가 options 페이지에서 직접 입력. `chrome.storage.sync`에 저장.

**가격:** 월 1000건 무료, 이후 $1.50/1000건.

---

## 15. 설정 페이지 (options.html)

포함 항목:
- API 키 입력 및 저장
- ESC 동작 선택 (전체 종료 / 모달만 닫기)
- 프라이버시 고지: "입력한 이미지는 OCR 처리를 위해 Google Cloud Vision API로 전송됩니다."

추후 확장 가능 항목 (지금은 UI 구조만 확보):
- 단축키 변경
- 언어 설정
- 복사 히스토리 on/off

---

## 16. 알려진 한계

- CSP가 엄격한 페이지에서 동작 불가 (구조적 한계)
- Shadow DOM 내부 이미지 미지원
- 회전 텍스트 오버레이 위치 어긋남 (추후 개선 예정)
- 서비스 워커 자동 종료: Vision API 응답이 1~3초 내이므로 실제 문제 없음
- `<all_urls>` 권한으로 인해 크롬 웹스토어 심사 시 추가 검토 대상이 될 수 있음
- URL 직접 전송 방식은 Google 서버가 해당 이미지 호스트에 의해 차단될 경우 폴백으로 전환됨

---

## 17. 추후 확장 가능 항목 (현재 미구현)

- 회전 텍스트 정확한 오버레이 지원
- 모달 내 확대/축소
- 이미지 회전
- 텍스트 번역
- 복사 히스토리
- 언어 선택 설정
- Chrome 웹스토어 배포

---

## 18. 개발 환경

- VSCode — 코드 편집
- GitHub Desktop — 버전 관리
- Chrome — 테스트 (chrome://extensions → 개발자 모드 → 압축해제된 확장 프로그램 로드)
- 코드 수정 후 chrome://extensions에서 새로고침(⟳) → 테스트 페이지 새로고침으로 반영
- 오류 확인: F12 → Console 탭 빨간 글씨
