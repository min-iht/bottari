# 시안 대조 스크린샷과 픽셀 UI 재현에서 만난 세 가지 함정

- 날짜: 2026-07-16
- 맥락: Bottari 리뉴얼 프로토타입(`.brainstorm/1815-1784207259/content/bottari-prototype.html`)을
  `draft/*.png` 시안(1320×2868 모바일 비율)과 대조·수정하는 visual-verdict 작업 중
- 태그: chrome-devtools-mcp, viewport-emulation, flexbox, min-width, css-grid

## 문제

시안과 같은 440×956 비율로 스크린샷을 찍어야 하는데 뷰포트가 504×690으로 잘렸고,
고정폭으로 지정한 요소(↵ 확인 버튼 52px, 열람 썸네일 카드 76px)가
실제 렌더링에서는 21px로 찌그러지거나 135px로 부풀었다. 둘 다 CSS 값만 봐서는 원인이 자명하지 않았다.

## 시도한 것들 (실패 포함)

1. `resize_page(440, 956)` → 뷰포트가 690 높이에서 잘림. 이 도구는 실제 창 크기를 바꾸는
   방식이라 모니터 물리 높이를 넘을 수 없다는 것을 확인 (창 리사이즈 가설 배제).
2. `.ok{ width:52px }` 지정 → flex 컨테이너에서 여전히 21px로 수축.
   width만으로는 flex-shrink를 이기지 못한다는 것을 확인.
3. 썸네일 `.icard{ width:76px }` → 긴 한글 텍스트가 든 카드가 135px로 팽창.
   flex item의 기본 `min-width:auto`(콘텐츠 최소폭)가 width보다 우선함을 확인.

## 통한 접근법

1. **뷰포트**: `emulate(viewport: "440x956x2,mobile,touch")` — 창 크기와 무관하게
   디바이스 에뮬레이션으로 원하는 해상도·DPR을 강제한다.
2. **수축 방지**: `flex:0 0 52px` (shrink 0 + 명시적 basis).
3. **팽창 방지**: `min-width:0` + `overflow:hidden` — 콘텐츠 최소폭 규칙을 해제.
4. (보너스) 아이템 3개를 위 1개·아래 2개로 배치: 2열 grid에
   `.icard:first-child:nth-last-child(3){ grid-column:1/3; justify-self:center; }` —
   "첫 번째이면서 뒤에서 3번째" = 총 3개일 때만 발동하는 조건부 스팬.

## 일반화된 교훈

- 스크린샷 뷰포트가 요청한 크기보다 작게 나오면 창 리사이즈 도구 대신 **emulate(디바이스 에뮬레이션)부터** 확인하라.
- flex 컨테이너 안 고정폭 요소가 **좁아지면 `flex:0 0 <px>`, 넓어지면 `min-width:0`** — 두 증상의 처방이 다르다.
- 개수에 따라 배치가 달라지는 그리드는 `:first-child:nth-last-child(n)` 조합으로 CSS만으로 분기할 수 있다.

## 재발 방지 (선택)

프로토타입에 `?clean` URL 파라미터로 개발 바를 숨기는 분기를 넣어 두었다 —
시안 대조 스크린샷은 항상 `?clean=1`로 찍으면 레이아웃 오염이 없다.
