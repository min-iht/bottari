# 시안 PNG와 렌더링 간격을 1:1로 맞추기 — 잉크 기준 측정과 웹폰트 로딩 레이스

- 날짜: 2026-07-17
- 맥락: Bottari 리뉴얼 프로토타입(`.brainstorm/1815-1784207259/content/bottari-prototype.html`) SCR-08 완료 화면을 시안 `draft/12.png`와 간격 동일하게 맞추는 작업
- 태그: visual-fidelity, pixel-scan, web-font, race-condition, css-spacing

## 문제

시안(1320×2868 PNG)과 프로토타입의 요소 간 간격을 "동일하게" 맞춰야 하는데,
눈대중 스크린샷 비교로는 몇 px 차이인지 알 수 없었다. 게다가
`getBoundingClientRect()`의 박스 간격과 시안에서 보이는 간격은 다르다 —
텍스트는 line-height 박스(20px) 밖으로 글자 잉크(29px 폰트)가 삐져나오고,
캐릭터 PNG는 배경이 불투명해 박스 전체가 잉크로 잡힌다.

## 시도한 것들 (실패 포함)

1. 박스(rect) 기준 간격 비교 → 시안은 픽셀 잉크 기준이라 기준이 어긋남.
   line-height(20px) < font-size(29px)인 제목에서 최대 ±5px 오차 발생 가능.
2. 수정 후 페이지 새로고침 직후 바로 측정 → dialog 높이가 139→191px로 튀었다.
   원인은 레이아웃 변화가 아니라 **웹폰트(WOFF2)가 아직 로드되지 않아 폴백 폰트로
   줄바꿈이 3줄→6줄로 늘어난 것**. `await document.fonts.ready`를 넣어도
   화면 전환(go()) 직후 새 폰트 로드가 시작되면 status가 다시 "loading"이 된다.

## 통한 접근법

양쪽을 모두 "잉크(ink)" 기준으로 측정해 대조했다.

- **시안 쪽**: PIL로 행별 콘텐츠 픽셀을 스캔해 요소별 밴드(y0–y1, x0–x1)를 추출.
  3x 해상도(1320px = 440 CSS px)이므로 ÷3으로 환산. 배경색은 모서리 픽셀로 잡고
  tol=8 정도로 낮게 — 옅은 눈송이 같은 저대비 콘텐츠가 tol=18에서는 누락됐다.
- **렌더링 쪽**: 박스는 `getBoundingClientRect()`, 텍스트 잉크는
  zero-size inline-block 프로브로 baseline 위치를 얻고 canvas
  `measureText().actualBoundingBoxAscent/Descent`를 더해 계산.
- **폰트 레이스**: 측정 전 `setTimeout(800ms)` 후 `document.fonts.ready`를 기다리고,
  결과 JSON에 `document.fonts.status === "loaded"`를 함께 반환해 유효성을 확인.
- 시안의 dialog 폭(340px)과 렌더링(339px)이 이미 일치하는 것으로 1:1 스케일임을
  먼저 검증한 뒤 간격 수정에 들어갔다 — 스케일 검증 없이 간격만 맞추면 헛수고다.

## 일반화된 교훈

1. **수치가 이유 없이 크게 튀면(레이아웃 +60px 등) 웹폰트 로드 상태부터 확인하라.**
   측정 JS에 `document.fonts.status`를 항상 포함시키면 오염된 측정을 즉시 걸러낸다.
2. **시안 대조는 박스가 아니라 잉크 기준으로.** line-height < font-size인 텍스트,
   투명/불투명 여백이 있는 이미지는 rect와 보이는 경계가 다르다.
3. **간격을 맞추기 전에 스케일 앵커를 먼저 찾아라.** 시안과 렌더링에서 이미 일치하는
   요소(여기선 dialog 폭)를 확인하면 환산 배율 가설이 검증된다.

## 재발 방지 (선택)

측정 스크립트(`scan12.py`류)를 쓸 때 tol을 8과 18 두 번 돌려 결과가 다르면
저대비 콘텐츠 누락을 의심한다. 브라우저 측정 스니펫은 fonts.status를 반환값에 포함.
