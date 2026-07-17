# 보따리(Bottari) Renewal — API 명세서
Version: 2.0
Date: 2026-07-17
상위 문서: `기능명세서.md` v2.0 (§1 archive.js · FN-17·18) · `erd.md` v2.0 · `권한정책.md` v2.0
다음 순서 문서: `qa_테스트_케이스.md`

---

## 0. 개요

외부 API는 **Supabase REST(PostgREST) 2개 엔드포인트**가 전부다. 호출 주체는
`js/archive.js`이며, 브라우저 `fetch()`로 직접 호출한다(SDK 미사용 — 노빌드 원칙).

- Base URL: `{SUPABASE_URL}/rest/v1` (`js/config.js`의 `SUPABASE_URL`)
- 모드 판정: `SUPABASE_URL`과 `SUPABASE_ANON_KEY`가 **둘 다** 비어 있지 않으면
  Supabase 모드, 하나라도 비면 localStorage 모드(§3 폴백 계약). 판정은 호출 시점마다
  config 상수를 읽어 수행한다(별도 상태 저장 없음).

### 공통 요청 헤더

| 헤더 | 값 |
|---|---|
| `apikey` | `{SUPABASE_ANON_KEY}` |
| `Authorization` | `Bearer {SUPABASE_ANON_KEY}` |
| `Content-Type` | `application/json` (POST만) |

## 1. 보관함 등록 — `POST /rest/v1/bottari_archive`

FN-17. SCR-10 "보따리 보관함에 맡기기" 클릭 시 1회 호출.

### 요청

```
POST {SUPABASE_URL}/rest/v1/bottari_archive
Prefer: return=representation
```

```json
{
  "name": "민경",
  "items": [
    { "type": "image", "imageData": "data:image/jpeg;base64,..." },
    { "type": "text", "textContent": "..." },
    { "type": "map", "mapLink": "https://...", "mapCaption": "..." }
  ],
  "owner_token_hash": "sha256-hex(ownerToken)"
}
```

- `owner_token_hash`: FN-17 3-1항의 `ownerToken`(UUID)을 `crypto.subtle.digest("SHA-256", ...)`로
  해시한 hex 문자열. 토큰 원문은 전송하지 않는다.

- `items`는 진행 상태의 items에서 내부 `id`를 제거한 배열(1~3개). `id`·`created_at`은
  보내지 않는다 — DB default가 채운다.
- `Prefer: return=representation`으로 생성된 행(uuid·created_at 포함)을 응답으로 받아,
  SCR-14 목록 맨 앞에 즉시 반영한다(재조회 없이).

### 응답

| 상태 | 의미 | 클라이언트 동작 |
|---|---|---|
| `201 Created` | 등록 성공. body = 생성된 행 배열(1건) | SCR-14로 전환, 등록분 맨 앞 표시 |
| 그 외 전부 (4xx/5xx/네트워크 오류/타임아웃) | 등록 실패 | 토스트 #14 "보관함에 맡기지 못했어요. 다시 시도해 주세요" + SCR-10 잔류, 카드 재활성화 |

- 실패 사유를 사용자에게 구분해 보여주지 않는다(문안표 §3.3이 정본 — 단일 문안).
  개발 진단용으로 `console.warn`에 상태 코드만 남긴다(콘솔 **에러** 0건 게이트 준수).

## 2. 보관함 목록 — `GET /rest/v1/bottari_archive`

FN-18. SCR-14 진입 시마다 호출.

### 요청

```
GET {SUPABASE_URL}/rest/v1/bottari_archive?select=id,name,items,created_at&order=created_at.desc
```

- 페이지네이션은 두지 않는다(포트폴리오 데모 규모). 항목이 늘면 `Range` 헤더 도입은
  후속 과제로 남긴다 — 현재 그리드는 행 추가로 전량 표시(FN-18 3항).

### 응답

| 상태 | 의미 | 클라이언트 동작 |
|---|---|---|
| `200 OK` | body = ArchiveEntry 배열 (최신순) | 그리드 렌더. 0건이면 빈 셀 12개 |
| 그 외 전부 | 조회 실패 | 토스트 #15 "보따리를 불러오지 못했어요. 다시 시도해 주세요" + 빈 셀 그리드 표시 |

- `select`에 `owner_token_hash`를 포함하지 않는다(표시에 불필요). 포함해 조회하더라도
  해시뿐이라 토큰은 노출되지 않는다(권한정책 §3.3).

## 2-1. 보관함 꺼내기 — `POST /rest/v1/rpc/remove_bottari`

FN-23. SCR-15의 [보관함에서 꺼내기] 확인 모달 통과 시 1회 호출.
REST DELETE 대신 토큰을 대조하는 `security definer` 함수 호출(권한정책 §3.3 — DELETE 정책 없음).

```
POST {SUPABASE_URL}/rest/v1/rpc/remove_bottari
```

```json
{ "p_id": "…uuid…", "p_token": "…ownerToken 원문…" }
```

| 상태 | 의미 | 클라이언트 동작 |
|---|---|---|
| `200 OK` + body `1` (삭제 행 수) | 꺼내기 성공 | `bottari_owned_v1`에서 제거 → SCR-14 재조회 |
| `200 OK` + body `0` | id 없음 또는 토큰 불일치 | 토스트 #16 + SCR-15 잔류 |
| 그 외 전부 | 호출 실패 | 토스트 #16 + SCR-15 잔류 |

- 함수 시그니처: `remove_bottari(p_id uuid, p_token text) returns integer` —
  `sha256(p_token)` = `owner_token_hash`인 행만 삭제하고 삭제 행 수를 반환. DDL은 `DB_SETUP.md`.

## 3. localStorage 폴백 계약 (`js/archive.js` 내부)

`archive.js`는 모드와 무관하게 동일한 두 함수를 노출한다. 호출부(app.js)는 모드를 모른다.

| 함수 | 반환 (Promise) | Supabase 모드 | localStorage 모드 |
|---|---|---|---|
| `Bottari.archive.register({ name, items })` | 성공: 생성된 entry / 실패: reject | §1 POST | `bottari_archive_v1` 배열 맨 앞 삽입. `id: crypto.randomUUID()`, `created_at`: ISO 문자열 |
| `Bottari.archive.list()` | 성공: entry 배열(최신순) / 실패: reject | §2 GET | `bottari_archive_v1` 파싱. 키 없음 → `[]`, 파싱 실패 → reject |
| `Bottari.archive.remove(id, ownerToken)` | 성공: resolve / 실패·불일치: reject | §2-1 RPC (반환 0이면 reject) | `bottari_archive_v1`에서 해당 id 제거 (없으면 reject) |

- 폴백 모드도 Promise를 반환해 호출부 코드가 분기하지 않게 한다.
- reject 시 호출부는 §1·§2와 동일하게 문안 #14/#15를 표시한다.

## 4. 내부 계약 — 공유 링크 포맷 (참고)

서버 API는 아니지만 URL이 곧 인터페이스이므로 형식을 고정한다(FN-15·16, crypto.js·sharelink.js 계승).

```
{origin}{pathname}?data={base64url( salt(16B) ∥ IV(12B) ∥ AES-GCM 암호문 )}
```

- 평문 페이로드: `{ "name": string, "items": [...] }` (id 제거본).
- 키 유도: PBKDF2-SHA256 150,000회 → AES-GCM 256. 전체 URL ≤ 2,000자.
- 복호화 실패 구분: base64url 해석 불가·길이 부족 → 손상(#13) / GCM 인증 실패 → 암호 오류(#11).
- 이 형식은 기존 배포본과 동일 코드(계승, 수정 없음)이므로 버전 필드를 두지 않는다.

## 5. 테이블·보안 전제

- 테이블 정의는 `erd.md` §2, RLS 정책은 `권한정책.md` §3.3이 정본이다.
  anon key로 허용되는 작업은 SELECT·INSERT와 RPC `remove_bottari` EXECUTE뿐이며,
  직접 UPDATE·DELETE는 정책 부재로 거부된다.
- Supabase 프로젝트 생성·키 발급·DDL 실행 절차는 `DB_SETUP.md`(구현 단계 작성)에 둔다.
