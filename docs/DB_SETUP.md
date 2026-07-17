# 보따리(Bottari) Renewal — DB_SETUP
Version: 2.0
Date: 2026-07-17
상위 문서: `erd.md` v2.0 (§2 테이블 정의) · `권한정책.md` v2.0 (§3.3 RLS) · `api_명세서.md` v2.0 (§1·2·2-1 호출 계약)

이 문서는 위 세 문서가 "DDL 원문과 설정 절차는 `DB_SETUP.md`"로 위임한 내용의 정본이다.
Supabase 없이도 앱은 localStorage 폴백으로 동작하므로(erd §3), 이 절차 전체는 **선택 사항**이다 —
공용 보관함을 켜고 싶을 때만 수행한다.

---

## 0. 준비물

- Supabase 계정 (https://supabase.com — 무료 티어로 충분, 포트폴리오 데모 규모)
- 소요 시간 약 10분. 모든 SQL은 대시보드의 SQL Editor에서 실행한다.

## 1. 프로젝트 생성과 키 발급

1. Supabase 대시보드 → **New project**.
   - Name: `bottari` (임의), Region: `Northeast Asia (Seoul)` 권장.
   - Database Password: 강한 비밀번호 생성 후 별도 보관 — 앱 코드에는 쓰이지 않는다.
2. 프로젝트 생성 완료 후 **Settings → API**에서 두 값을 복사한다.
   - `Project URL` → `SUPABASE_URL`
   - `anon` `public` 키 → `SUPABASE_ANON_KEY`
3. `js/config.js`에 붙여넣는다.

```js
// js/config.js
const SUPABASE_URL = "https://xxxxxxxxxxxx.supabase.co";
const SUPABASE_ANON_KEY = "eyJ...";
```

- 두 값 중 하나라도 빈 문자열이면 앱은 localStorage 모드로 동작한다(api_명세서 §0 모드 판정).
- **`service_role` 키는 절대 복사하지 않는다** — 코드·문서·커밋 어디에도 두지 않는다(권한정책 §3.3).
  관리 작업(스팸 정리 등)은 대시보드에서 수동으로 한다.

## 2. DDL — 테이블·인덱스

SQL Editor에서 아래를 그대로 실행한다. 컬럼 정의의 정본은 erd.md §2와 동일하다.

```sql
-- 보관함 테이블 (유일한 관계형 테이블)
create table public.bottari_archive (
  id               uuid        primary key default gen_random_uuid(),
  name             text        not null,
  items            jsonb       not null,
  created_at       timestamptz not null default now(),
  owner_token_hash text        not null
);

-- 목록 조회(created_at 내림차순 — FN-18) 정렬용 인덱스
create index idx_bottari_archive_created_at
  on public.bottari_archive (created_at desc);
```

- `items` 내부 구조는 DB에서 검증하지 않는다 — 스키마 정본은 기능명세서 §2.2 (erd §2 주석).
- `owner_token_hash`에는 토큰의 SHA-256 hex만 저장한다. 토큰 원문은 어디에도 저장하지 않는다.

## 3. RLS — 활성화와 정책

**RLS 활성화는 배포 전 필수 게이트다**(권한정책 §3.3). anon key가 공개되는 정적 사이트이므로,
RLS가 꺼진 상태로 배포하면 누구나 임의 수정·삭제가 가능해진다.

```sql
-- RLS 활성화 (정책이 없는 작업은 전부 거부됨)
alter table public.bottari_archive enable row level security;

-- 공개 읽기 — 보관함은 모든 방문자 공용 (권한정책 §3.3)
create policy archive_public_read
  on public.bottari_archive
  for select
  to anon
  using (true);

-- 공개 등록 — 등록은 누구나 (권한정책 §3.3)
create policy archive_public_insert
  on public.bottari_archive
  for insert
  to anon
  with check (true);

-- UPDATE / DELETE 정책은 만들지 않는다 = 전면 거부.
-- 삭제는 §4의 RPC remove_bottari로만 수행한다.
```

## 4. RPC — 꺼내기 함수 `remove_bottari`

꺼내기(FN-23)는 REST DELETE가 아니라 소유 토큰을 대조하는 `security definer` 함수로만 한다
(api_명세서 §2-1). `sha256()`은 PostgreSQL 내장 함수라 확장 설치가 필요 없다.

```sql
create or replace function public.remove_bottari(p_id uuid, p_token text)
returns integer
language plpgsql
security definer
set search_path = public
as $$
declare
  v_deleted integer;
begin
  delete from public.bottari_archive
   where id = p_id
     and owner_token_hash = encode(sha256(convert_to(p_token, 'UTF8')), 'hex');
  get diagnostics v_deleted = row_count;
  return v_deleted;  -- 1 = 삭제 성공, 0 = id 없음 또는 토큰 불일치
end;
$$;

-- anon이 함수만 호출할 수 있게 권한 정리
revoke all on function public.remove_bottari(uuid, text) from public;
grant execute on function public.remove_bottari(uuid, text) to anon;
```

- 클라이언트는 토큰 **원문**을 이 RPC에만 보낸다(api_명세서 §2-1). 함수가 서버 측에서
  해시해 대조하므로, anon key가 노출돼도 토큰 없이는 남의 보따리를 지울 수 없다.
- 반환값(삭제 행 수)이 `0`이면 클라이언트는 실패로 처리한다(토스트 #16).

## 5. 검증 체크리스트

SQL Editor에서 순서대로 확인한다. 전부 통과해야 배포 게이트를 넘는다
(qa_테스트_케이스.md의 보관함 케이스와 연동).

```sql
-- (1) RLS가 켜져 있는가 → rowsecurity = true
select relname, relrowsecurity from pg_class where relname = 'bottari_archive';

-- (2) 정책이 SELECT·INSERT 2개뿐인가
select policyname, cmd from pg_policies where tablename = 'bottari_archive';

-- (3) INSERT → 삭제 왕복이 동작하는가
insert into public.bottari_archive (name, items, owner_token_hash)
values ('검증', '[{"type":"text","textContent":"test"}]',
        encode(sha256(convert_to('test-token', 'UTF8')), 'hex'))
returning id;  -- 반환된 id를 아래에 사용

select public.remove_bottari('여기에-반환된-id', 'wrong-token'); -- 기대: 0
select public.remove_bottari('여기에-반환된-id', 'test-token');  -- 기대: 1
```

앱 레벨 확인 (브라우저):

- [ ] `config.js`에 두 값 입력 후 SCR-10에서 "보관함에 맡기기" → SCR-14 목록 맨 앞에 표시 (FN-17)
- [ ] 다른 브라우저(또는 시크릿 창)에서 보관함 진입 → 같은 항목이 보임 (공용 확인)
- [ ] 등록한 브라우저의 SCR-15에서 [보관함에서 꺼내기] → 목록에서 사라짐 (FN-23)
- [ ] 다른 브라우저에서는 꺼내기 버튼 자체가 노출되지 않음 (`bottari_owned_v1` 부재)
- [ ] REST로 직접 DELETE 시도 시 거부되는가 (선택 — 콘솔에서):
  `fetch(SUPABASE_URL + "/rest/v1/bottari_archive?id=eq.…", { method: "DELETE", headers: { apikey: SUPABASE_ANON_KEY, Authorization: "Bearer " + SUPABASE_ANON_KEY } })`
  → 응답에 삭제 행이 없어야 정상 (RLS 거부)

## 6. 운영 메모

- 남용(스팸 INSERT, 전량 SELECT)은 위험 수용 — 필요 시 대시보드의 rate limit이 완화 수단
  (권한정책 §3.3). 부적절 콘텐츠 삭제는 대시보드 Table Editor에서 수동으로 한다.
- 무료 티어는 장기간 미사용 시 프로젝트가 일시정지(pause)될 수 있다 — 데모 시연 전
  대시보드에서 프로젝트가 Active 상태인지 확인한다.
- 테이블 구조 변경 시 이 문서와 `erd.md` §2를 함께 갱신한다 (컬럼 정의 정본은 erd.md).
