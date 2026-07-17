# GitHub Pages 프로젝트 사이트에서 location.origin으로 만든 공유 URL은 404가 난다

- 날짜: 2026-07-17
- 맥락: Bottari 리뉴얼 프로토타입(`index.html`) SCR-12 "주소 알려주기"가 생성한 공유 URL을 새 창에서 열면 404. 배포처는 GitHub Pages 프로젝트 사이트(`min-iht.github.io/bottari/`)
- 태그: github-pages, location-origin, url-routing, spa, deep-link

## 문제

로컬(파일/localhost 루트)에서는 멀쩡하던 공유 URL이 배포본에서만 404. 코드가
`location.origin + '/?data=' + mock`으로 URL을 만들고 있었는데, 프로젝트 사이트의
`origin`은 `https://min-iht.github.io`까지만이라 리포 경로 `/bottari/`가 통째로
빠진다. 루트 도메인에는 사용자 사이트가 없으므로 404. 로컬에서는 루트가 곧
앱이라 증상이 재현되지 않아 원인이 바로 보이지 않았다.

## 시도한 것들 (실패 포함)

1. SCR-13 마크업 손상 의심 — grep 출력에 `<\div>`, `src="img\..."` 같은 깨진 조각이
   보였다 → Read로 원본 바이트를 확인하니 정상. grep 결과 표시가 뭉개진 것이었다.
   (교훈: 도구 출력의 이스케이프 아티팩트를 소스 손상으로 착각하지 말 것)
2. 라우팅 부재 의심 → 이건 실제로 두 번째 층의 버그였다. URL 경로를 고쳐도
   부트 시 `go('scr01')` 고정이라 `?data=`로 들어온 수신자에게 SCR-13이 뜨지 않는다.
   404는 경로 문제, "SCR-13이 안 뜸"은 라우팅 문제 — 원인이 두 개였다.

## 통한 접근법

- URL 생성: `location.origin + location.pathname + '?data=' + mock` — 현재 문서의
  실제 경로를 기준으로 만들면 루트 배포·하위 경로 배포 어디서든 유효하다.
- 부트 라우팅: `go(new URLSearchParams(location.search).has('data') ? 'scr13' : 'scr01')`
  — 파라미터 검사는 `search.includes()`가 아니라 `URLSearchParams.has()`로.
  (`includes('dev')`류는 랜덤 페이로드 문자열에 우연히 매칭될 수 있다)

## 일반화된 교훈

1. 배포본에서만 링크가 404면 **URL을 어떻게 조립했는지**부터 확인하라 —
   `location.origin`은 하위 경로 배포(GitHub Pages 프로젝트 사이트, 리버스 프록시)를
   모른다. 절대 URL이 필요하면 `origin + pathname`을 쓴다.
2. "공유 링크가 안 열린다"는 증상은 보통 두 층이다: ① 링크 자체가 유효한 경로인가,
   ② 열렸을 때 앱이 그 진입점을 인식하는가. 하나 고치고 끝내지 말고 둘 다 검증하라.
3. URL 파라미터 검사는 `location.search.includes()` 대신 `URLSearchParams.has()` —
   부분 문자열 오탐을 구조적으로 막는다.

## 재발 방지 (선택)

배포 대상이 하위 경로일 수 있는 프로젝트에서는 공유·딥링크 URL을 만드는 지점에
"루트 기준 절대경로 금지" 주석을 남겼다(SCR-12 생성부에 반영).
