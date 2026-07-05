# Drawing Diary — 개발 기술 스펙

작성일: 2026-07-06
단계: 3단계 개발 (웹 프로토타입)
기준 문서: brief.md, PRD.md, design-context.md, wireframe.md

---

## 출력물

- `index.html` 단일 파일 (HTML + CSS + JS 모두 인라인)
- 외부 의존: Lucide Icons CDN 1개만 허용

---

## 기술 스택

| 항목 | 결정 |
|------|------|
| 언어 | HTML5 / CSS3 / Vanilla JS (ES6+) |
| 빌드 도구 | 없음 |
| 외부 라이브러리 | Lucide Icons (CDN) |
| 데이터 저장 | IndexedDB (localStorage 사용 금지) |
| 폰트 | 시스템 폰트 스택만 (`-apple-system, BlinkMacSystemFont, 'Noto Sans KR', sans-serif`) |

---

## IndexedDB 스키마

- DB명: `drawing-diary-db`
- 스토어명: `entries`
- 키: `"YYYY-MM-DD"` (예: `"2026-07-06"`)
- 레코드:
  ```json
  {
    "date": "2026-07-06",
    "canvasPng": "<canvas.toDataURL('image/png') 결과>",
    "objects": []
  }
  ```
- `objects`는 MVP에서 빈 배열 고정 (사진·텍스트는 캔버스에 합성 후 PNG로 통합 저장)

---

## 화면 구조 (단일 페이지)

```
┌─────────────────────┐
│  상단 바 (48px)      │  읽기: 날짜 바 / 편집: 편집 바
├─────────────────────┤
│                     │
│  캔버스 영역 (flex)  │  남은 높이 전체
│                     │
├─────────────────────┤
│  하단 바 (56px)      │  읽기: 액션 바 / 편집: 도구 모음
└─────────────────────┘
```

---

## 구현할 기능 (PRD Must Have)

| 기능 | 구현 방법 |
|------|-----------|
| F01 캔버스 그리기 | `<canvas>` + `pointerdown/move/up` 이벤트. `devicePixelRatio` 대응. |
| F02 사진 붙이기 | `<input type="file" accept="image/*">`. 선택 이미지를 canvas 위 오브젝트로 삽입. MVP에서 앨범 선택만 지원 (카메라 직접 촬영 제외). |
| F03 텍스트 메모 | canvas 위 `<div contenteditable>` 오버레이 입력 → blur 시 `element.textContent`로 값 추출 → `fillText()`로 canvas에 렌더링. XSS 방지: `innerHTML` 파싱 금지. |
| F04 날짜 기반 페이지 | 오늘 날짜로 시작. 이미 저장된 날짜면 편집 모드 진입. 하루 한 페이지 고정. |
| F05 페이지 넘기기 | 좌우 버튼 + 스와이프 (임계값 50px). 편집 모드 중 스와이프 비활성. |
| F06 로컬 저장 | IndexedDB에 PNG + 오브젝트 저장. 저장 버튼 탭 시 즉시 저장. |
| F07 이미지 내보내기 | `canvas.toDataURL()` → `<a download>` 트리거. 공유 시트(bottom sheet)에서 실행. |
| F08 실행취소 (Undo) | 획 배열(strokes[]) 마지막 항목 제거 후 캔버스 재렌더링. |
| F09 지우개 | 지우개 도구 선택 시 `destination-out` 합성 모드로 그리기. |

---

## 앱 상태 모델

```
mode: 'read' | 'edit'
currentDate: 'YYYY-MM-DD'
tools: 'pen' | 'eraser' | 'text' | 'image'
penColor: '#1A1A1A'   (기본값)
penSize: 4            (기본값, px)
strokes: []           (현재 세션 획 배열 — undo용)
objects: []           (사진·텍스트 오브젝트 배열)
drawerOpen: boolean   (색상·굵기 서랍 열림 여부)
```

---

## 인터랙션 규칙 (design-context.md 요약)

- 모든 터치 가능 요소: 최소 44×44px 실효 터치 영역
- 취소 동작:
  - 변경 없음 → 즉시 읽기 모드 복귀
  - 변경 있음 → "변경 사항을 버릴까요?" 확인 다이얼로그 표시
    - "버리기" → 마지막 저장본 복원 후 읽기 모드
    - "계속 편집" → 그대로 유지
- 저장 성공: "저장되었습니다" 토스트 2초
- 저장 실패: "저장에 실패했습니다. 브라우저 저장공간을 확인해주세요" 빨간 토스트 4초
- 한글 IME 가드: Enter `keydown`에 `if (e.isComposing) return;`
- 스와이프 원위치: ease-out 120ms

---

## 디자인 토큰 (CSS 변수로 선언)

```css
--color-bg:           #FAF8F5;
--color-surface:      #FFFFFF;
--color-canvas-bg:    #FEFCF9;
--color-text-primary: #1A1A1A;
--color-text-sub:     #8A8A8A;
--color-border:       #E5E0D8;
--color-disabled:     #C4BFB8;
--color-accent:       #6B4F3A;
--color-danger:       #D94F3D;

/* 드로잉 팔레트 */
--sw1: #1A1A1A; --sw2: #6B6B6B; --sw3: #B87870;
--sw4: #C09070; --sw5: #C4AF70; --sw6: #7BAA8A;
--sw7: #7090B8; --sw8: #9080B8;
```

타이포: 날짜 15px/400, 버튼 13px/500~600, 안내문구 12px/400  
레이아웃: 상단바 48px, 하단바 56px, 화면 패딩 16px, 버튼 radius 8px, 시트 radius 16px

---

## 반응형

- 기준: 375px (iPhone SE 기준)
- 캔버스 논리 크기: `window.innerWidth × (window.innerHeight - 104)`
- canvas 실제 픽셀: 논리 크기 × `devicePixelRatio` (Retina 대응)

---

## 아이콘

- Lucide Icons CDN. `lucide.createIcons({ attrs: { 'stroke-width': 1 } })`
- 크기: UI 아이콘 18×18px, 빈 캔버스 아이콘 28×28px
- CDN 실패 시: 텍스트 레이블 대체 (↩ / 펜 / 지우개 / T / 사진)
