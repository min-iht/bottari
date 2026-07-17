# mermaid: 같은 노드 쌍의 다중 엣지는 렌더에서 라벨이 조용히 누락된다

- 날짜: 2026-07-16
- 맥락: Bottari 리뉴얼 문서 작업 중 `docs/화면흐름도.md` 수신자 흐름 flowchart 검증 (D:\claude_code\vibe\Bottari)
- 태그: mermaid, flowchart, self-loop, validation, chrome-devtools

## 문제

화면흐름도의 mermaid 5개 블록을 mermaid@11의 `parse()` + `render()`로 검증해 5 PASS를 받았는데, 렌더된 SVG에는 SCR-13 self-loop 2개(틀린 암호 / 페이로드 손상) 중 "틀린 암호" 엣지 라벨이 없었다. 파싱·렌더 모두 예외 없이 성공했기 때문에 PASS/FAIL 결과만 봐서는 누락을 알 수 없었다.

## 시도한 것들 (실패 포함)

1. `mermaid.parse()`로 문법 검증 → PASS. 그러나 문법 통과가 렌더 완전성을 보장하지 않는다는 것이 이 문제의 본질이었다 (라벨 누락은 예외를 던지지 않고 조용히 발생).
2. Browser pane(`mcp__Claude_Browser__navigate`)으로 검증 HTML 열기 → `file://`와 `http://localhost` 모두 "denied or failed"로 거부. 이 surface로는 로컬 검증 페이지를 열 수 없음을 확인하고 2회 실패 시점에 경로를 바꿨다.
3. chrome-devtools MCP(`new_page` → `take_snapshot`)로 열어 a11y 트리 확인 → "틀린 암호" 텍스트 부재 발견. `evaluate_script`로 SVG의 `.edgeLabel` 전수 조사 → 같은 노드 쌍(S13→S13) 엣지 2개 중 하나만 라벨이 렌더됨을 확정.

## 통한 접근법

같은 노드로 향하는 self-loop 2개를 엣지 1개로 합치고(라벨 "틀린 암호 / 페이로드 손상"), 케이스별 문안 구분은 다이어그램 아래 불릿으로 옮겼다. 재렌더 후 `.edgeLabel` 목록을 다시 뽑아 두 라벨이 모두 존재함을 확인했다.

검증 하네스(재사용 가능): md에서 ` ```mermaid ` 블록을 PowerShell 정규식으로 추출 → `blocks.js` → CDN mermaid@11로 parse+render하는 HTML → `python -m http.server` → chrome-devtools `new_page`로 열어 summary와 `.edgeLabel` 확인.

## 일반화된 교훈

1. mermaid 검증은 "파싱 PASS"가 아니라 "렌더된 SVG에 의도한 엣지 라벨이 전부 있는가"까지 봐야 끝난다 — 라벨 누락은 예외를 던지지 않는다.
2. 같은 노드 쌍을 잇는 엣지(특히 self-loop)를 2개 이상 선언하면 라벨 하나가 조용히 사라진다 — 엣지를 1개로 합치고 케이스 구분은 본문 불릿으로 뺀다. GitHub 렌더러도 같은 mermaid라 동일하게 재현된다.
3. 이 워크스페이스에서 로컬 HTML을 브라우저로 검증할 때는 Browser pane navigate(`file://`·`localhost` 거부됨)가 아니라 chrome-devtools MCP의 `new_page`를 쓴다.

## 재발 방지 (선택)

문서 품질 게이트에서 mermaid 검사 시 parse 결과와 함께 SVG `.edgeLabel` 텍스트 목록을 소스의 `|"라벨"|` 개수와 대조하면 이 유형이 구조적으로 잡힌다.
