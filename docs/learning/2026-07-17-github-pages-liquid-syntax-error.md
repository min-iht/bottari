# GitHub Pages 빌드 실패 — docs 마크다운의 `{{` 가 Jekyll Liquid 문법 오류를 일으킴

- 날짜: 2026-07-17
- 맥락: Bottari 리뉴얼(`D:\claude_code\vibe\Bottari`)을 GitHub Pages(main 브랜치, `/` 루트)로 배포 설정. 원인 파일은 `docs/qa_테스트_케이스.md` 103행.
- 태그: github-pages, jekyll, liquid, markdown, static-site

## 문제

GitHub Pages에서 브랜치를 main으로 설정했는데 사이트 URL이 생성되지 않았다. 저장소 설정 화면에는 아무 에러도 보이지 않았고, Pages API(`gh api repos/<owner>/<repo>/pages`)를 조회해야 `status: errored`가 드러났다. 빌드 API의 에러 메시지도 "Page build failed."라는 일반 문구뿐이라 원인이 자명하지 않았다.

## 시도한 것들 (실패 포함)

1. Pages 빌드 API(`pages/builds/latest`)에서 에러 메시지 확인 → "Page build failed."만 반환. 구체 원인 없음.
2. 저장소 크기(14MB)·대용량 파일·루트 구조 점검 → 모두 정상. 용량/구조 가설 배제.
3. 순수 HTML 사이트이므로 `.nojekyll` 파일 추가 후 push → **여전히 Jekyll이 실행되며 실패.** 자동 "pages build and deployment" 워크플로우(`actions/jekyll-build-pages`)는 이 저장소에서 `.nojekyll`을 무시했다.
4. `gh run view <run-id> --log-failed`로 Actions 실패 로그 직접 열람 → 진짜 원인 발견: `Liquid Exception: Variable '{{' was not properly terminated ... in docs/qa_테스트_케이스.md` (103행).

## 통한 접근법

QA 문서에서 "손상 데이터" 예시로 쓴 `` `"{{"` `` 문자열이 원인이었다. Jekyll은 마크다운 렌더링 전에 Liquid 템플릿 파서를 먼저 돌리는데, 이때 `{{`를 변수 시작으로 해석하고 닫는 `}}`가 없어 빌드 전체가 죽는다. 코드 백틱(`` ` ``) 안에 있어도 소용없다 — Liquid가 마크다운보다 먼저 실행되기 때문이다. 예시를 `"{"`(홑괄호, 여전히 잘못된 JSON이라 테스트 의도 보존)로 바꿔 push하자 빌드가 즉시 성공했다.

## 일반화된 교훈

1. **GitHub Pages URL이 안 생기면 설정 화면 말고 로그부터**: `gh api repos/<o>/<r>/pages`로 `status` 확인 → `errored`면 `gh run list` + `gh run view <id> --log-failed`로 Actions 로그를 직접 읽는다. Pages API의 에러 메시지는 항상 일반 문구다.
2. **Pages에 올라가는 마크다운/HTML에 `{{` 또는 `{%`가 있으면 빌드가 죽을 수 있다**: 배포 전 `grep -rE '\{\{|\{%'`로 스캔한다. 템플릿 예시·손상 데이터 예시·mustache 문법 인용이 흔한 범인이다.
3. **`.nojekyll`이 항상 Jekyll을 꺼주지는 않는다**: 자동 워크플로우가 `actions/jekyll-build-pages`를 돌리는 저장소에서는 `.nojekyll`이 무시될 수 있다. 확실한 해결은 원인 문자를 제거하거나, GitHub Actions 커스텀 워크플로우(`upload-pages-artifact`)로 정적 업로드하는 것이다.

## 재발 방지 (선택)

배포 대상 저장소에 Liquid 위험 문자가 새로 들어오는지 커밋 전 `grep -rE '\{\{|\{%' --include='*.md' --include='*.html'` 한 번이면 잡힌다. 문서에 `{{`를 꼭 써야 하면 `{% raw %}...{% endraw %}`로 감싼다.
