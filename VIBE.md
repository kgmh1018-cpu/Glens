# VIBE 2.0 — Allow-Copy Design & Interaction Reference

> v1에서 빠진 "왜 프리미엄하게 느껴지는가"의 실체를 담은 버전.
> AI 프롬프트용 한 문단 요약은 맨 아래.

---

## 한 줄 무드

> "애플 디자인팀이 만든 미니멀 유틸리티 앱. 모든 인터랙션이 물리적 촉감처럼 느껴진다."

---

## 1. 컬러 & 타이포 (v1과 동일)

| 역할 | 값 |
|---|---|
| 배경 | `#f5f5f7` |
| 카드/모달 | `#ffffff` |
| 주 잉크 | `#1d1d1f` |
| 보조 잉크 계층 | `#3c3c3c` → `#6e6e73` → `#8e8e93` → `#aeaeb2` → `#d0d0d0` |
| 강조 | `#007aff` (Apple Blue 단 하나) |
| 성공 | `#32D74B` (토스트 전용) / `#28a745` (텍스트 전용) |
| 경고 | `#FF9F0A` |

폰트: `Plus Jakarta Sans` — 자간 항상 음수, border `0.5px` hairline만.

---

## 2. 애니메이션 시스템 — 이게 핵심

### CSS transition으로 안 되는 것들은 rAF으로 직접 짠다

이 프로젝트의 프리미엄함은 CSS만으론 불가능해요.
커서 이동, 드래그 ghost 경로, 텍스트 하이라이트 진행 — 전부 `requestAnimationFrame` 루프로 직접 구현.

**직접 구현한 easing 함수 목록 (그대로 복붙해서 쓰세요):**

```js
function easeOutQuart(t)    { return 1 - Math.pow(1 - t, 4); }
function easeOutCubic(t)    { return 1 - Math.pow(1 - t, 3); }
function easeInOutSine(t)   { return -(Math.cos(Math.PI * t) - 1) / 2; }
function easeHumanDrag(t)   { return t < 0.5 ? 2 * t * t : 1 - Math.pow(-2 * t + 2, 2) / 2; }
function easeOutBack(t)     { const c1 = 1.70158; return 1 + (c1+1)*Math.pow(t-1,3) + c1*Math.pow(t-1,2); }
function easeOutBackSoft(t) { const c1 = 0.6;     return 1 + (c1+1)*Math.pow(t-1,3) + c1*Math.pow(t-1,2); }
function easeInCubic(t)     { return t * t * t; }
function easeOutExpo(t)     { return t === 1 ? 1 : 1 - Math.pow(2, -10 * t); }
```

**Bezier 곡선 경로 보간 패턴 (드래그 ghost 이동):**

```js
function animDrag(el, fromX, fromY, toX, toY, dur, onDone) {
  const dx = toX - fromX, dy = toY - fromY;
  // 제어점: 초반 12%는 천천히, 후반 58%부터 가속
  const cp1x = fromX + dx * 0.12, cp1y = fromY + dy * 0.04;
  const cp2x = fromX + dx * 0.78, cp2y = fromY + dy * 0.58;
  let prevX = fromX;
  const start = performance.now();
  function tick(now) {
    const raw = Math.min((now - start) / dur, 1);
    const t = easeOutQuart(raw), mt = 1 - t;
    const x = mt*mt*mt*fromX + 3*mt*mt*t*cp1x + 3*mt*t*t*cp2x + t*t*t*toX;
    const y = mt*mt*mt*fromY + 3*mt*mt*t*cp1y + 3*mt*t*t*cp2y + t*t*t*toY;
    // 이동 방향에 따른 자연스러운 기울기 (±5도)
    const tilt = Math.max(-5, Math.min(5, (x - prevX) * 0.65));
    el.style.left = x + 'px';
    el.style.top  = y + 'px';
    el.style.transform = `scale(1) rotate(${tilt}deg)`;
    prevX = x;
    if (raw < 1) requestAnimationFrame(tick);
    else if (onDone) onDone();
  }
  requestAnimationFrame(tick);
}
```

---

## 3. 인터랙션 연출 철학 — "인간적인 타이밍"

### 사용자를 관찰하고 공감하고 해결한다

이 앱의 데모 애니메이션은 단순 시연이 아니라 **사용자가 겪는 좌절을 직접 연기**해요.

```
1. 텍스트 드래그 시도 (실패)        550ms 동안 끝까지 긁음
2. 더블클릭으로 강제 선택 시도      재빠른 복귀 + 2연타 클릭
3. 좌절해서 마우스 흔들기           frustratedJiggle keyframe (450ms)
4. Allow-Copy 버튼으로 이동         확신에 찬 Bezier 곡선 이동
5. 버튼 꾹 눌림 + 드래그 픽업       scale(0.88) 후 spring 복원
6. 북마크바로 드래그                ghost 기울기 + 드롭 indicator
7. 드롭 + 클릭 실행                 버튼 click 촉감
8. 텍스트 하이라이트 성공 (피날레)  I-beam 커서로 확신있게 스윕
9. 감상 (300ms 정지)                체류 시간 의도적 확보
10. 커서 퇴장                       scale(0.8) + fadeout으로 소멸
```

**핵심 원칙: 각 단계 사이의 공백이 연출이다**
- 실패 후 멈추는 120ms → 당황하는 느낌
- 드롭 성공 후 300ms 감상 → 사용자가 결과를 인식하는 시간
- 커서가 `easeInCubic`으로 화면 밖으로 빠짐 → 갑자기 사라지지 않는 자연스러운 퇴장

---

## 4. 버튼 촉감 시스템 — 3단계 상태

```
기본    → floatBtn (3s ease-in-out infinite) + shadow
눌림    → scale(0.96) translateY(1px) + shadow 축소  [0.1s]
복원    → springBack keyframe (0.95→1.02→1)           [0.4s, easeOutBack]
```

```css
@keyframes springBack {
  0%   { transform: scale(0.95); opacity: 0.5; }
  50%  { transform: scale(1.02); opacity: 1; }
  100% { transform: scale(1);    opacity: 1; }
}
```

**클릭 시 잘못된 상태(wiggle):**
```css
@keyframes wiggle {
  0%, 100% { transform: translateX(0) rotate(0); }
  20%       { transform: translateX(-6px) rotate(-1.5deg); }
  40%       { transform: translateX(5px) rotate(1.5deg); }
  60%       { transform: translateX(-3px) rotate(-0.5deg); }
  80%       { transform: translateX(2px) rotate(0.5deg); }
}
```

**`void el.offsetWidth` 패턴** — 같은 animation을 재실행할 때 reflow 강제 필수:
```js
btn.classList.remove('wiggle');
void btn.offsetWidth;  // 이 한 줄 없으면 브라우저가 애니메이션 생략
btn.classList.add('wiggle');
```

---

## 5. 토스트 컴포넌트 — 가장 정교한 컴포넌트

```css
background: rgba(28, 28, 30, 0.88);
backdrop-filter: blur(20px);
-webkit-backdrop-filter: blur(20px);
border: 0.5px solid rgba(255, 255, 255, 0.12);  /* 흰 hairline으로 입체감 */
border-radius: 999px;
padding: 8px 16px 8px 8px;
font-size: 13px; font-weight: 500;
color: #fff;
letter-spacing: -0.01em;
```

**아이콘 circle:**
```css
width: 24px; height: 24px;
border-radius: 50%;
background: #32D74B;  /* 성공 / 경고: #FF9F0A */
```

**등장/퇴장 (translateY + opacity 복합):**
```js
// 시작 상태: translateX(-50%) translateY(-10px), opacity: 0
// rAF 더블 프레임 — 이렇게 안 하면 transition이 적용 안 됨
requestAnimationFrame(() => {
  requestAnimationFrame(() => {
    toast.style.opacity = '1';
    toast.style.transform = 'translateX(-50%) translateY(0)';
  });
});
// 퇴장: 다시 translateY(-10px) + opacity:0, 250ms 후 remove()
```

**체류 시간:**
- 데모: 1400ms
- 실제 실행: 800ms (방해 최소화)

---

## 6. 모달 뷰 전환 — 높이 애니메이션

```js
function switchView(views, nextView, className) {
  views.style.height = views.offsetHeight + 'px';  // 현재 높이 고정
  views.classList.add(className);                    // 뷰 전환 트리거
  requestAnimationFrame(() => {
    views.style.height = nextView.scrollHeight + 'px';  // 목표 높이
  });
  views.addEventListener('transitionend', function onEnd(e) {
    if (e.propertyName !== 'height') return;
    views.style.height = '';  // 자연 높이로 복원
    views.removeEventListener('transitionend', onEnd);
  });
}
```

슬라이드 easing: `cubic-bezier(0.32, 0.72, 0, 1)` — iOS sheet 느낌.

---

## 7. Optimistic UI 패턴

```
클릭 즉시  → 성공 UI 전환 (체크 아이콘 spring 등장)
백그라운드 → fetch fire-and-forget (실패해도 UI 안 바뀜)
1000ms 후  → 메인 뷰로 슬라이드 복귀
400ms 후   → 폼 조용히 리셋 (전환 끝난 뒤)
```

체크 아이콘 spring:
```css
@keyframes drawCheck {
  0%   { opacity: 0; transform: scale(0.5); }
  70%  { transform: scale(1.1); }
  100% { opacity: 1; transform: scale(1); }
}
animation: drawCheck 0.35s cubic-bezier(0.34, 1.56, 0.64, 1) forwards;
```

---

## 8. 성공 순간 연출 — shimmer + cardPulse

```css
@keyframes shimmerUnlock {
  0%   { background-position: 120% center; }
  100% { background-position: -20% center; }
}
.shimmer-text {
  background: linear-gradient(90deg, #1d1d1f 40%, #ffffff 50%, #1d1d1f 60%);
  background-size: 300% auto;
  color: transparent !important;
  -webkit-background-clip: text;
  background-clip: text;
  animation: shimmerUnlock 0.55s cubic-bezier(0.4, 0, 0.2, 1) forwards;
}

@keyframes cardPulse {
  0%   { transform: scale(1);    box-shadow: 0 12px 32px rgba(0,0,0,0.07); }
  20%  { transform: scale(0.97); box-shadow: 0 4px 12px rgba(0,0,0,0.04); }
  50%  { transform: scale(1.03); box-shadow: 0 16px 48px rgba(0,0,0,0.12); }
  100% { transform: scale(1);    box-shadow: 0 12px 32px rgba(0,0,0,0.07); }
}
```

---

## 9. drag ghost 세부

```css
background: rgba(29, 29, 31, 0.88);
color: #f5f5f7;
border-radius: 10px;
backdrop-filter: blur(10px);
box-shadow: 0 16px 32px rgba(0,0,0,0.2), 0 4px 12px rgba(0,0,0,0.1);

/* 픽업: scale(0.95) rotate(-2deg) → spring으로 scale(1.04) */
/* 드롭: scale(0.85) translateY(2px) + opacity:0 */
/* 이동 중: 방향에 따라 rotate(±5deg) 자동 적용 */
```

---

## 10. 모바일 swipe-to-close

```
dragMode: 'none' | 'drag' | 'scroll'
3px 이상 움직임 감지 후 의도 판별:
  scrollTop <= 0 && dy > 0 → 'drag' 모드
  그 외 → 'scroll' 모드 (기본 스크롤 허용)
drag 모드: modal translateY(max(0, dy)) 실시간 반영
손 뗄 때 dy > 80 → 닫힘 / 그 외 → spring 복귀
touchmove passive: false 필수 (preventDefault 필요)
```

---

## 11. 쓰지 않는 것

- 원색 (토스트 아이콘 `#32D74B`, `#FF9F0A` 외)
- `1px` 이상 border (focus ring 제외)
- 과한 그라디언트 (스크롤 마스크만)
- 스크롤바 노출
- setTimeout 기반 위치 애니메이션 (rAF 필수)
- 여러 easing을 같은 용도에 혼용

---

## AI 프롬프트용 한 문단 요약 (v2 — 이걸 붙여넣으세요)

```
디자인: Apple HIG 라이트모드 미니멀. 배경 #f5f5f7, 카드 흰색 border-radius 22px,
주 버튼 #1d1d1f 다크. 폰트 Plus Jakarta Sans, 자간 항상 음수(-0.01~-0.03em).
border 0.5px hairline만. 강조색 #007aff 하나. 쉐도우 3단 레이어.

인터랙션: hover/상태변화는 CSS transition. 위치·경로 애니메이션은 rAF 루프로 직접 구현.
easing: 감속은 easeOutQuart, spring은 easeOutBack(c1=1.70158), 가속퇴장은 easeInCubic.
버튼 촉감 3단계: float → scale(0.96) 눌림 → springBack(0.95→1.02→1) 복원.
같은 animation 재실행 시 void el.offsetWidth로 reflow 강제 필수.

토스트: rgba(28,28,30,0.88) + blur(20px) + 흰border 0.5px, pill shape.
translateY(-10px)+opacity:0 → 0+1 등장. rAF 더블프레임 사용.

모달: spring(0.34,1.3,0.64,1) 등장. 뷰 전환은 height 고정→목표→auto 3단계.
모바일 bottom sheet + dragMode 판별 swipe-to-close.

연출: 사용자 좌절 공감 → 해결 스토리라인. 단계 사이 공백이 감정 만든다.
성공 시 shimmer + cardPulse. Optimistic UI로 즉각 반응 후 백그라운드 처리.
```
