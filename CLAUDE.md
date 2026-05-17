# GLens — CLAUDE.md

Chrome 확장 프로그램. 웹페이지 이미지 클릭 → OCR → 텍스트 선택/복사.
Manifest V3 / 순수 JS / 외부 프레임워크 없음.

---

## 파일 구조 & 로드 순서

```
manifest.json   host_permissions, content_scripts 로드 순서 정의
background.js   Service Worker. Vision API 호출, 이미지 fetch
content.js      진입/종료 총괄. 반드시 마지막 로드
modal.js        모달 UI
overlay.js      텍스트 오버레이 (PDF.js 방식)
panel.js        선택 텍스트 실시간 표시 패널
content.css     모든 스타일. manifest에서 자동 주입
options.html/js 설정 페이지
```

**manifest.json content_scripts 로드 순서 — 절대 변경 금지:**
```json
"js": ["overlay.js", "panel.js", "modal.js", "content.js"]
```
content.js가 나머지 세 파일의 함수를 모두 사용하므로 반드시 마지막.

---

## 전역 네임스페이스

모든 파일 간 통신은 `window._glens` 단일 객체로만 한다.

```javascript
window._glens = {
  active: false,
  currentRequest: null,
  enter() {},
  exit() {},       // DOM 제거 + null 할당 + GC 유도 필수
  modal:   { open, close, showLoading, showError, showNoText },
  overlay: { render, clear },
  panel:   { show, hide, update },
}
```

---

## FORBIDDEN — 절대 하지 말 것

**🚫 style 태그 직접 주입 금지**
```javascript
// 금지
document.createElement('style') → innerHTML 주입

// 올바른 방법
document.body.classList.add('_glens-active')
// content.css에서 ._glens-active * { cursor: zoom-in !important }
```

**🚫 tabId 없는 배지 조작 금지**
```javascript
// 금지 — 모든 탭에 배지가 적용되는 버그
chrome.action.setBadgeText({ text: 'ON' })

// 올바른 방법
chrome.action.setBadgeText({ text: 'ON', tabId: sender.tab.id })
```

**🚫 background.js에서 일반 canvas 금지**
Service Worker는 DOM이 없다. 반드시 OffscreenCanvas 사용.

**🚫 CSS 클래스명에 _glens- 접두사 누락 금지**
페이지 CSS와 충돌 방지. 모든 클래스는 `_glens-`로 시작.

**🚫 exit() 호출 시 메모리 해제 누락 금지**
modal DOM 제거 + OffscreenCanvas 초기화 + base64 변수 null 할당 필수.

---

## 이미지 처리 흐름

```
클릭된 요소 파악
  img 태그         → currentSrc || src  (currentSrc 우선)
  background-image → CSS에서 URL 파싱
  canvas           → toDataURL()
  파일 업로드      → FileReader → base64

URL 케이스:
  1순위 → Vision API에 URL 직접 전송 (전처리 없음)
  실패 시 폴백 → background.js fetch → 파이프라인 → base64

base64 케이스:
  → background.js 파이프라인 → Vision API
```

**파이프라인 (base64 필요 시):**
```
Step 1. 너비 > maxWidth → maxWidth로 축소
Step 2. JPEG 85% 인코딩 → 7MB 이하면 전송 (대부분 여기서 끝)
Step 3. 안전망: 너비를 minWidth까지 축소 + 85% 재인코딩 → 전송
        품질은 85% 고정. 해상도로만 크기 조절.
```

**IMAGE_PROCESSING 상수 — background.js 상단에 위치:**
```javascript
const IMAGE_PROCESSING = {
  quality:         0.85,
  maxWidth:        1600,
  minWidth:        800,
  maxPayloadBytes: 7 * 1024 * 1024,
};
```
수치 변경 시 반드시 이 객체만 수정. 코드 곳곳에 하드코딩 금지.

---

## Vision API

- 기능: `DOCUMENT_TEXT_DETECTION`
- 응답: `response.textAnnotations[1..n]` — 단어별 boundingPoly.vertices 포함
- JSON 요청 한도: 10MB / base64 실질 상한: 7.3MB
- API 키: `chrome.storage.sync`에서 로드

**메시지 구조:**
```javascript
// → background.js
{ action: 'processImage', imageUrl: '...', requestId: 'abc' }  // URL 케이스
{ action: 'processImage', imageData: '...', requestId: 'abc' } // base64 케이스

// → content.js
{ action: 'ocrResult', words: [...], imageData: '...', requestId: 'abc' }
{ action: 'ocrError', error: 'fetch_failed' | 'ocr_failed', requestId: 'abc' }
```

---

## 텍스트 오버레이 좌표 계산

```javascript
// img.onload 완료 후에만 실행할 것 (로드 전 실행 시 좌표 전부 0)
const scaleX = img.getBoundingClientRect().width  / img.naturalWidth
const scaleY = img.getBoundingClientRect().height / img.naturalHeight

// 회전 텍스트
const v = boundingPoly.vertices  // [좌상단, 우상단, 우하단, 좌하단]
const angle = Math.atan2(v[1].y - v[0].y, v[1].x - v[0].x)
span.style.transform = `rotate(${angle}rad)`
span.style.transformOrigin = 'top left'
```

---

## 선택 텍스트 패널

- `selectionchange` 이벤트 → `document.getSelection().toString()` → `panel.update(text)`
- 복사: `navigator.clipboard.writeText()`
- 모달과 독립된 별도 DOM. 모달 닫힐 때 함께 제거.

---

## manifest.json 필수 항목

```json
{
  "manifest_version": 3,
  "host_permissions": ["<all_urls>"],
  "content_scripts": [{
    "css": ["content.css"],
    "js": ["overlay.js", "panel.js", "modal.js", "content.js"]
  }]
}
```
