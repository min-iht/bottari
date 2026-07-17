# 백그라운드 탭에서 video play()가 차단될 때 — play 스텁 + 시킹으로 연출 로직 검증

- 날짜: 2026-07-17
- 맥락: Bottari 리뉴얼 프로토타입(`index.html`) SCR-09 엔딩 영상에 "마지막 1초 검정 fade out"을 추가한 뒤, 브라우저 자동화 팬에서 실제 재생으로 검증하려던 중
- 태그: autoplay-policy, background-tab, video, browser-automation, css-transition

## 문제

엔딩 영상 fade out을 검증하려고 SCR-09에 진입시키면 영상이 재생되지 않고 곧바로 다음 화면(SCR-10)으로 넘어갔다. 원인이 자명하지 않았던 이유: `play()` 거부가 기존 폴백(`catch → go('scr10')`, FN-12 재생 실패 스킵)에 **삼켜져서**, 겉으로는 "영상 없이 화면이 넘어감"으로만 보였다. 에러도 콘솔에 남지 않았다.

## 시도한 것들 (실패 포함)

1. `go('scr09')`를 JS로 직접 호출 후 opacity 폴링 → currentTime이 0에서 멈춤. "제스처가 없어서"라고 추정했으나 muted 영상은 원래 제스처 없이도 재생 가능하므로 이 가설은 불완전.
2. 실제 클릭(computer 도구)으로 GO! 버튼을 눌러 제스처 부여 → 그래도 즉시 다음 화면으로 튕김. **제스처 부족 가설 배제.**
3. `play()`에 catch 계측을 심고 재시도 → 진짜 원인 확보: `AbortError: The play() request was interrupted because video-only background media was paused to save power`. 자동화 팬은 백그라운드 탭 취급이라 **muted(video-only) 미디어 재생 자체를 절전 정책으로 차단**한다. 환경 제약이므로 우회 불가.

## 통한 접근법

재생 없이 실제 코드 경로를 검증했다.

```js
v.play = () => Promise.resolve();  // 절전 차단 우회 — startEnding()이 정상 진입
go('scr09');                       // opacity 리셋·핸들러 등록 등 실제 경로 실행
v.currentTime = 7.5;               // 시킹도 timeupdate를 발화 → 마지막 1초 fade 트리거 확인
v.dispatchEvent(new Event('ended')); // ended → 다음 화면 배선 확인
```

시킹이 `timeupdate`를 발화시키므로 재생 없이도 시간 기반 트리거를 검증할 수 있고, 스텁 덕에 프로덕션 코드는 한 줄도 바꾸지 않았다. 단, 백그라운드 탭은 렌더링이 스로틀되어 CSS transition의 **중간 프레임**까지는 확인 불가 — style 값·transition 선언·배경색 확인으로 갈음했다.

## 일반화된 교훈

1. "영상이 안 나오고 화면만 넘어간다" 증상이 보이면, 먼저 `play()` 반환 promise에 계측 catch를 심어 **거부 사유 문자열부터 확보**하라. 폴백이 잘 설계된 코드일수록 원인이 조용히 삼켜진다.
2. 자동화 브라우저 팬에서 muted 영상이 재생되지 않으면 제스처 문제가 아니라 **백그라운드 탭 절전 정책**("video-only background media")부터 의심하라. 실사용(포그라운드 탭)에서는 재현되지 않는 자동화 전용 제약이다.
3. 시간 기반 미디어 연출은 재생 없이도 검증 가능하다: `play()` 스텁 + `currentTime` 시킹(→ timeupdate 발화) + 이벤트 dispatch로 트리거·배선을 실제 코드 경로 그대로 태워라.
