# 음원이 작으면 재생 측에서 증폭하지 말고 파일을 정규화하라

- 날짜: 2026-07-17
- 맥락: Bottari 리뉴얼 `index.html`의 BGM(FN-22). 음원 `bgm/bottari_bgm.mp3`가 표준보다 한참 작게 들렸다.
- 태그: audio, ffmpeg, loudnorm, web-audio-api, autoplay-policy, lufs

## 문제

BGM을 켜도 음량이 표준보다 눈에 띄게 작았다. 원인이 자명하지 않았던 이유는 증상이 두 겹이었기 때문이다.
사운드 기본값이 OFF라 "설정이 안 됐다"로도, "음량이 작다"로도 읽혔다. 게다가 이전 세션이 이미
"음원이 약 20dB 작음"을 인지하고 Web Audio `GainNode` ×12 + `DynamicsCompressor` 리미터로
**재생 시점에** 증폭하는 체인을 넣어둔 상태였다 — 즉 "이미 고쳐진 것처럼 보이는" 코드가 있었다.

## 시도한 것들 (실패 포함)

1. 코드부터 읽고 "기본값 false가 원인" 으로 결론 → 기본값만 바꿨다면 음량 문제는 그대로 남았다.
   증상 두 개가 한 문장에 섞여 들어온 상황에서 하나만 보고 닫을 뻔한 케이스.
2. 기존 GainNode 체인을 신뢰하고 배수만 키우기 → 배제. 코드 주석의 근거 수치(peak -22.4dBFS, RMS -38.7dB)가
   **RMS 기준**이라 웹 표준(LUFS)과 직접 비교가 안 됐다. 추측 대신 측정이 필요했다.
3. `ffmpeg -af loudnorm=print_format=json`으로 실측 → **-35.9 LUFS**. 표준 -16 LUFS 대비 20 LU 낮음이 확정됐다.
   이 시점에 "재생 측 증폭"이 아니라 "파일이 잘못됐다"로 문제가 재정의됐다.
4. `loudnorm` 기본(dynamic) 모드 → 배제. 루프 BGM에 동적 압축이 걸리면 강약이 뭉개진다(LRA 9.0 → 8.5).
   `linear=true`로 전환하니 상수 게인만 적용돼 LRA 9.0 → 9.1로 원곡 강약이 보존됐다.
5. (2차 요청 "좀 더 크게") `loudnorm:linear=true`로 -14.5 LUFS 시도 → **실패**. `linear=true`를 줘도 요청 게인이
   TP 한도를 넘으면 ffmpeg가 경고 없이 **Dynamic으로 조용히 폴백**한다(출력의 `Normalization Type`에만 드러난다).
   loudnorm만으로 linear을 유지할 수 있는 상한은 `TP목표 - measured_TP`가 정하며, 여기서는 -15.3 LUFS였다 —
   체감이 거의 없는 +1.0 LU라 요청을 만족시키지 못했다.
6. `volume` 상수 게인 + `alimiter`(look-ahead 트루피크 리미터) → **성공**. loudnorm의 dynamic이 장시간 게인을
   출렁이게 하는 것과 달리, 리미터는 최상단 트랜지언트만 깎아 LRA가 9.00으로 **원본과 완전히 동일**하게 남았다.

## 통한 접근법

파일 자체를 정규화하고, 재생 측 증폭 체인을 통째로 삭제했다. 최종적으로 쓴 것은 loudnorm이 아니라
**상수 게인 + 리미터** 조합이다.

```bash
# 1) 실측 — 게인 계산의 근거
ffmpeg -i in.mp3 -af loudnorm=I=-14:TP=-1:LRA=11:print_format=json -f null -
#    → input_i=-35.94, input_tp=-22.39, input_lra=9.00

# 2) 목표 - 실측 = 필요 게인. -14.2 LUFS를 노리면 +22dB.
#    alimiter가 트루피크만 잡아주므로 크레스트가 큰 음원도 linear 폴백 없이 목표에 도달한다.
#    limit=0.85 ≈ -1.4 dBFS (mp3 인코딩 시 피크가 조금 올라오는 여유분)
ffmpeg -i in.mp3 -af "volume=22dB,alimiter=limit=0.85:attack=5:release=50:level=disabled" \
  -ar 44100 -b:a 192k out.mp3
# 결과: -14.2 LUFS / -1.7 dBTP / LRA 9.00 (원본 LRA와 동일)
```

이게 통한 이유는 **문제를 발생 지점에서 고쳤기 때문**이다. GainNode 체인은 증상을 재생 시점에 덧칠하는
방식이라 (a) `createMediaElementSource`는 `file://`처럼 origin이 opaque한 환경에서 무음이 될 수 있고,
(b) AudioContext resume·리미터 튜닝 같은 상태가 늘고, (c) 음원을 교체할 때마다 배수를 다시 맞춰야 한다.
파일이 표준 라우드니스면 `<audio>` 태그만으로 끝나고 이 셋이 전부 사라진다.

그리고 **상수 게인 + 리미터가 loudnorm보다 나은 이유**는 게인이 시간에 따라 움직이지 않기 때문이다.
loudnorm의 dynamic 모드는 구간별로 게인을 조정하므로 루프 BGM에서는 곡의 시작과 끝 게인이 달라져
**이음매에서 음량이 튈 수 있다**. 리미터는 최상단 트랜지언트만 순간적으로 깎아 본체를 건드리지 않는다.

곁가지 두 개도 같이 정리했다. 기본값 ON은 autoplay 정책상 부팅 즉시 재생이 불가능하므로,
첫 `pointerdown`/`keydown`에서 재생하고 성공 시 리스너를 해제하는 방식으로 처리했다.

## 일반화된 교훈

1. **"음량이 작다/크다" 증상이 보이면 재생 코드를 만지기 전에 `ffmpeg loudnorm print_format=json`으로
   LUFS부터 측정하라.** 웹/스트리밍 기준은 -14 LUFS(Spotify·YouTube), 방송 계열은 -16 LUFS다.
   dBFS peak나 RMS 수치는 체감 음량과 직접 대응하지 않으니 그 값으로 게인 배수를 추정하지 마라.
2. **에셋이 규격 미달이면 에셋을 고쳐라. 재생 측 보정은 최후의 수단이다.** 런타임 보정은 환경 의존
   실패 모드(`file://` opaque origin, autoplay, AudioContext 상태)를 새로 만들고 교체 비용을 영구히 남긴다.
3. **루프 BGM을 크게 만들 때는 loudnorm이 아니라 `volume` 상수 게인 + `alimiter`를 써라.**
   loudnorm은 `linear=true`를 줘도 게인이 TP 한도를 넘으면 **경고 없이 dynamic으로 폴백**한다
   (`Normalization Type` 출력으로만 확인 가능). dynamic은 게인이 시간에 따라 움직여 루프 이음매에서 음량이 튄다.
   상수 게인 + 리미터는 최상단 트랜지언트만 깎아 LRA를 원본과 동일하게 남긴다.
4. **크레스트 팩터(TP - I)가 정규화 상한을 결정한다.** 상수 게인만으로 도달 가능한 최대 라우드니스는
   `목표TP - 실측TP + 실측I`다. 이 선을 넘고 싶으면 반드시 피크를 깎아야 하니, "얼마나 크게"는
   음질과의 교환이다 — 임의로 정하지 말고 들을 사람에게 물어라.
5. **"이미 고쳐둔 것처럼 보이는 코드"의 주석에 적힌 근거 수치를 그대로 믿지 마라.** 재측정이 30초면 끝난다.
   여기서는 주석의 근거가 RMS라 기준 자체가 어긋나 있었다.
6. **재정규화는 항상 원본에서 다시 하라.** 이미 정규화된 파일을 또 인코딩하면 mp3 세대 손실이 누적된다.

## 재발 방지

- `docs/기능명세서.md` FN-22-6과 `docs/DESIGN.md` §9에 "음원은 -14 LUFS로 정규화해서 넣는다
  (재생 측 볼륨 보정 없음, 상수 게인 + 리미터, TP -1 dBTP 이하)"를 명시 — 음원 교체 시 규격으로 걸린다.
- `docs/qa_테스트_케이스.md`에 QA-72b(스트리밍 표준 음량 청취, 클리핑 없음) 추가.
- 원본 음원은 커밋 `5798472`에 보존 — 정규화가 과했다면 `git show 5798472:bgm/bottari_bgm.mp3`로 복구.
