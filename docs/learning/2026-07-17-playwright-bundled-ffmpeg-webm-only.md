# Playwright 번들 ffmpeg으로는 mp4를 만들 수 없다 (VP8/WebM 전용 빌드)

- 날짜: 2026-07-17
- 맥락: `D:\claude_code\vibe\Bottari` 보따리 프로토타입(index.html) 전체 흐름을 playwright-cli로 촬영해 mp4 데모 영상으로 납품하려던 작업. `page.screencast.start()`는 WebM(VP8)만 출력하므로 mp4 변환 단계가 필요했다.
- 태그: ffmpeg, playwright, video-encoding, windows, h264

## 문제

시스템에 ffmpeg이 없었다. 그런데 Playwright가 영상 녹화용으로 자체 ffmpeg 바이너리를 이미 내려받아 두고 있었다(`ms-playwright/ffmpeg-1011/ffmpeg-win64.exe`). "ffmpeg이 이미 있으니 변환도 이걸로 하면 된다"가 자연스러운 결론이지만 틀렸다. 파일명·실행 방식·버전 출력이 정품 ffmpeg과 똑같아서, 실제로 변환을 걸어보기 전까지는 못 쓴다는 사실이 드러나지 않는다.

## 시도한 것들 (실패 포함)

1. 시스템 ffmpeg 탐색 (`where ffmpeg`, choco/scoop/winget, `C:\ffmpeg`, python `imageio_ffmpeg`) → 전부 없음. 설치 없이는 해결 불가라는 것만 확인.
2. Playwright 번들 ffmpeg 사용 시도 → `-encoders`로 확인해 보니 **인코더가 `libvpx`(VP8) 하나뿐**, muxer도 `webm` 하나뿐. `libx264` 없음, mp4 muxer 없음. 녹화에 필요한 최소 기능만 담은 스트립 빌드였다. 변환을 걸었다면 "Unknown encoder 'libx264'"로 죽었을 것.
3. cua-driver의 `install_ffmpeg` 툴 → dry-run 결과 `choco install ffmpeg -y`를 제안. Chocolatey는 있었지만 `C:\ProgramData`에 쓰므로 관리자 권한 필요 → 비권한 셸에서 실패 위험.

## 통한 접근법

`npm i -g ffmpeg-static` — 사용자 폴더(`AppData\Roaming\npm`)에만 쓰므로 관리자 권한이 필요 없고, gyan.dev essentials 빌드(ffmpeg 6.1.1, libx264 포함)를 그대로 가져온다. 8초 만에 설치 완료.

```bash
FF="C:/Users/rebec/AppData/Roaming/npm/node_modules/ffmpeg-static/ffmpeg.exe"
"$FF" -i demo.webm -c:v libx264 -preset slow -crf 20 \
  -pix_fmt yuv420p -movflags +faststart -an out.mp4
```

`-pix_fmt yuv420p`(구형 플레이어 호환)와 `-movflags +faststart`(moov 아톰을 파일 선두로 → 웹 스트리밍 즉시 재생)가 납품용 mp4의 필수 옵션이다.

## 일반화된 교훈

1. **번들된 도구는 "그 도구의 풀버전"이 아니다.** 라이브러리가 끼워 넣은 바이너리(ffmpeg, node, python 등)를 본래 용도 밖으로 쓰기 전에 능력부터 조회하라 — ffmpeg이면 `-encoders`/`-muxers`/`-decoders`. 이름이 같다고 기능이 같지 않다.
2. **Windows에서 CLI 도구가 필요한데 관리자 권한이 불확실하면 npm 전역 설치를 먼저 검토하라.** `AppData\Roaming\npm`은 사용자 쓰기 권한이라 UAC를 타지 않는다. choco/winget은 시스템 경로에 써서 비권한 셸에서 실패한다. `ffmpeg-static`, `7zip-bin` 등 바이너리 래퍼 패키지가 이 용도로 존재한다.
3. **"파일이 생성됐다"는 검증이 아니다.** 변환 후 `ffmpeg -v error -i out.mp4 -f null -` 로 전체 디코드해 오류 0건을 확인하고, 프레임을 뽑아 실제 화면이 담겼는지 눈으로 봐라. 크기만 보고 넘어가면 깨진 파일을 납품한다.

## 재발 방지

이 PC에는 이제 `C:\Users\rebec\AppData\Roaming\npm\node_modules\ffmpeg-static\ffmpeg.exe`에 전체 기능 ffmpeg이 있다. 다음에 영상 변환이 필요하면 재설치하지 말고 이 경로를 쓴다. (PATH에는 없으므로 절대경로 호출 필요.)
