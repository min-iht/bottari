# 모바일 첫 진입이 느릴 때 — 범인은 이미지 하나가 아니라 "대역폭 경쟁"이다

- 날짜: 2026-07-17
- 맥락: Bottari 리뉴얼(`D:\claude_code\vibe\Bottari\index.html`). 모바일에서 이미지가 늦게 뜨고
  랜딩 텍스트가 돋움체(대체 폰트)로 먼저 보였다가 바뀌는 FOUT 발생.
- 태그: webp, preload, font-display, bandwidth, mobile-performance

## 문제

데스크톱에서는 멀쩡한데 모바일에서만 이미지 로드가 눈에 띄게 늦고 폰트가 번쩍였다.
이전 커밋에서 `document.fonts.load()` 폰트 예열까지 넣었는데도 FOUT이 남아,
"폰트 로딩 코드"만 봐서는 원인이 자명하지 않았다.

## 시도한 것들 (실패 포함)

1. 폰트 예열(`document.fonts.load`, 이전 커밋) → 숨은 화면 폰트 FOUT은 잡았지만
   랜딩 첫 진입 FOUT은 못 잡음. 예열은 `<body>` 끝 스크립트라 시작 자체가 늦고,
   CDN 연결 수립(DNS+TLS) 비용은 그대로였다. → "예열 코드 유무"가 아니라 "시작 시점·연결 비용"이 문제.
2. 이미지 무손실 WebP 변환 시도 → 1328KB가 934KB로 30%만 감소. 픽셀아트풍이라도
   텍스처가 섞인 일러스트는 무손실로는 안 줄어든다. → 손실 q85~90으로 전환.

## 통한 접근법

세 가지를 한 세트로 처리했다. 핵심은 **첫 화면에 필요한 바이트만 먼저, 나머지는 로드 완료 후**.

1. **PNG → 손실 WebP(q90)**: img/ 6.3MB → 266KB (원본 PNG 보존, 새 .webp 파일로 출력).
   3배 확대 비교로 픽셀 열화 없음 확인. `image-rendering:pixelated`와도 무관.
2. **무거운 미디어의 preload 강등**: BGM 1.4MB·엔딩 영상 1.6MB가 `preload="auto"`로
   첫 페인트와 대역폭을 경쟁하고 있었다 → `preload="none"` + `window load` 이벤트에서
   `.load()`로 예열. 화면 전환 시 src가 바뀌는 이미지들도 같은 타이밍에 `new Image()`로 예열.
3. **폰트 preconnect + preload**: `<head>`에 `preconnect`(CDN 연결 선수립)와
   랜딩 화면에 실제로 쓰이는 폰트 3종만 `<link rel="preload" as="font" crossorigin>`.
   (전 폰트를 preload하면 그것끼리 또 경쟁한다 — 첫 화면에 보이는 폰트만.)

## 일반화된 교훈

- "모바일에서만 느리다"는 증상이 보이면 코드보다 먼저 **네트워크 워터폴의 총량과 순서**를 확인하라.
  `preload="auto"`인 오디오/비디오가 있으면 그게 1순위 용의자다.
- 표시 크기(150px)와 원본 해상도(1268px)의 배율이 3배(레티나 감안)를 넘으면 용량 낭비다.
  일러스트·픽셀아트는 무손실이 아니라 **손실 WebP q85~90**부터 시도하라 — 확대 비교로 검증하면 된다.
- 폰트 FOUT은 `font-display`나 JS 예열만으로 안 끝난다. CDN 폰트라면
  `preconnect` + 첫 화면 폰트만 `preload`가 정공법이다. preload는 crossorigin 필수(없으면 이중 다운로드).

## 재발 방지 (선택)

새 이미지 추가 시 체크: 표시 폭×3(레티나) 이상의 원본이면 WebP로 변환해 넣는다(품질 기준 300KB 이하).
`preload="auto"` 미디어 태그는 추가 금지 — `preload="none"` + load 후 예열 패턴을 쓴다.
