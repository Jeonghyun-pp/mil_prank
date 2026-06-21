# 사진 아카이브 켜기 — Supabase 설정 (5분, 무료)

설정 안 해도 사이트는 **로컬 모드**로 잘 돌아갑니다(카운트다운/퍼센트/놀림). 아래를 따라하면 **공유 모드**가 켜져서 누구나 사진을 올리고 친구별 앨범이 모두에게 공유돼요.

---

## 1) Supabase 프로젝트 만들기
1. https://supabase.com 가입 (GitHub 계정으로 1초)
2. **New project** → 이름/비밀번호 아무거나, 리전은 **Northeast Asia (Seoul)** 추천
3. 프로젝트 생성까지 1~2분 대기

## 2) 키 복사
좌측 **Project Settings(톱니) → API** 에서 두 개를 복사:
- **Project URL** → `https://xxxx.supabase.co`
- **anon public** key → `eyJ...` (긴 문자열, 공개돼도 안전한 키)

## 3) index.html 에 붙여넣기
`index.html` 상단 스크립트의 이 두 줄을 채워주세요:
```js
var SUPABASE_URL = "https://xxxx.supabase.co";  // ← Project URL
var SUPABASE_ANON_KEY = "eyJ...";               // ← anon public key
```

## 4) 데이터베이스 만들기 (SQL 한 번 실행)
좌측 **SQL Editor → New query** 에 아래를 통째로 붙여넣고 **Run**:

```sql
-- 친구 명단
create table if not exists public.people (
  id text primary key,
  name text not null,
  enlist date not null,
  months int not null default 21,
  created_at timestamptz not null default now()
);

-- 사진
create table if not exists public.photos (
  id uuid primary key default gen_random_uuid(),
  person_id text not null references public.people(id) on delete cascade,
  url text not null,
  path text not null,
  nickname text,
  caption text,
  created_at timestamptz not null default now()
);

-- 리액션
create table if not exists public.reactions (
  id uuid primary key default gen_random_uuid(),
  photo_id uuid not null references public.photos(id) on delete cascade,
  emoji text not null,
  created_at timestamptz not null default now()
);

-- 최형우 기본 등록 (2026-06-22 입대, 공군 21개월)
insert into public.people (id, name, enlist, months)
values ('choi', '최형우', '2026-06-22', 21)
on conflict (id) do nothing;

-- 사진 저장 버킷 (공개)
insert into storage.buckets (id, name, public)
values ('photos', 'photos', true)
on conflict (id) do nothing;

-- RLS 켜기
alter table public.people enable row level security;
alter table public.photos enable row level security;
alter table public.reactions enable row level security;

-- 읽기: 누구나
create policy "read people"    on public.people    for select using (true);
create policy "read photos"    on public.photos    for select using (true);
create policy "read reactions" on public.reactions for select using (true);

-- 추가: 누구나 (완전 공개)
create policy "add people"     on public.people    for insert with check (true);
create policy "add photos"     on public.photos    for insert with check (true);
create policy "add reactions"  on public.reactions for insert with check (true);

-- 삭제
create policy "del people"     on public.people    for delete using (id <> 'choi'); -- 최형우는 삭제 불가
create policy "del reactions"  on public.reactions for delete using (true);          -- 리액션 토글용

-- 사진 파일 업로드/읽기 (anon)
create policy "upload photo files" on storage.objects for insert with check (bucket_id = 'photos');
create policy "read photo files"   on storage.objects for select using (bucket_id = 'photos');
```

> 한 번 실행 후 "policy already exists" 류 에러가 나면, 이미 만들어진 것이니 무시해도 됩니다.

## 5) 끝!
`index.html` 새로고침 → 상단 배지가 **"공유 모드 ☁️"** 로 바뀌고, **＋ 사진 올리기** 버튼이 동작합니다.

---

## 관리(부적절한 사진 삭제)
완전 공개라 누구나 올릴 수 있어요. 이상한 사진이 올라오면:
- Supabase 대시보드 **Table Editor → photos** 에서 해당 행 삭제
- **Storage → photos** 버킷에서 실제 파일 삭제

## 용량
- 무료 티어 저장소 **1GB**. 사진은 업로드 시 자동으로 최대 1600px/JPEG로 줄여서 저장하니 수천 장은 거뜬해요.

## 배포
설정 끝났으면 `index.html` 을 그대로 정적 호스팅(Netlify Drop / Vercel / GitHub Pages)에 올리면 됩니다. 서버 불필요.
