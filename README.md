# Wakeup Clock ⏰🌅

매일 아침 **06:00 KST**에 GitHub Actions(cron)가 Supabase `wakeup` 테이블의
`created_at`을 현재 시각으로 갱신하고, `index.html`이 그 기록을 **아날로그 시계**로 보여줍니다.

> 무료 Supabase 프로젝트가 비활성으로 일시정지되는 것을 막는 "하트비트" 용도로도 유용합니다.

## 파일 구조
```
wakeup-clock/
├─ .github/workflows/wakeup.yml    # 매일 06:00 KST 실행 (cron)
├─ .github/workflows/cleanup.yml   # 매일 09:00 KST, 오래된 run 정리 (최신 10개 유지)
├─ .github/workflows/deploy.yml    # main 푸시 시 GitHub Pages 자동 배포
├─ index.html                      # 아날로그 시계 (GitHub Pages에 배포됨)
├─ .env.example                    # 환경변수 템플릿 (복사해서 .env로 사용)
└─ README.md
```

## 1. Supabase 준비
1. `wakeup` 테이블에 **id = 1** 행이 있어야 합니다.
2. 시계 페이지가 읽을 수 있도록 `wakeup` 테이블에 **SELECT 허용 RLS 정책**을 추가합니다.
   SQL Editor에서:
   ```sql
   alter table public.wakeup enable row level security;

   create policy "public read wakeup"
   on public.wakeup
   for select
   to anon
   using (true);
   ```
   (쓰기는 워크플로가 service_role 키로 RLS를 우회해서 처리하므로, 쓰기 정책은 필요 없습니다.)

## 2. GitHub Secrets 등록

저장소 → **Settings → Secrets and variables → Actions → New repository secret**

### 2-1. 워크플로 공통 / wakeup.yml 전용

| Secret 이름 | 값 | 확인 위치 |
|---|---|---|
| `SUPABASE_URL` | `https://<프로젝트>.supabase.co` | Project Settings → API → Project URL |
| `SUPABASE_SERVICE_ROLE_KEY` | service_role 키 | Project Settings → API → service_role |

> ⚠️ `service_role` 키는 **Secrets에만** 넣고, 절대 공개 코드에 넣지 마세요.

### 2-2. 시계 페이지 배포용 (deploy.yml 전용)

| Secret 이름 | 값 | 확인 위치 |
|---|---|---|
| `SUPABASE_ANON_KEY` | anon (public) 키 | Project Settings → API → anon public |

> `anon` 키는 공개돼도 되는 키입니다. Secrets 대신 **Variables**에 넣어도 됩니다.  
> Variables를 쓰는 경우 `deploy.yml`에서 `secrets.SUPABASE_ANON_KEY` →  
> `vars.SUPABASE_ANON_KEY`로 변경하세요.

### Secrets 등록 단계별 안내

```
1. 저장소 상단 탭 → Settings 클릭
2. 왼쪽 사이드바 → Secrets and variables → Actions 클릭
3. [New repository secret] 버튼 클릭
4. Name 입력 (예: SUPABASE_URL)
5. Secret 입력 (실제 값)
6. [Add secret] 클릭
7. 위 표의 나머지 Secret도 동일하게 반복
```

등록 후 목록에서 이름만 보이고 값은 다시 볼 수 없는 것이 정상입니다.

## 3. GitHub Pages 활성화

1. 저장소 → **Settings → Pages**
2. **Source** → **GitHub Actions** 선택
3. 저장

이후 `main` 브랜치에 푸시하거나 **Actions → Deploy to GitHub Pages → Run workflow**를 실행하면 자동 배포됩니다.

## 4. 동작 흐름

```
push to main
    └─▶ deploy.yml 실행
            ├─ index.html의 __SUPABASE_URL__ → secrets.SUPABASE_URL 치환
            ├─ index.html의 __SUPABASE_ANON_KEY__ → secrets.SUPABASE_ANON_KEY 치환
            └─ GitHub Pages에 배포
                └─▶ https://<유저명>.github.io/<저장소>/

매일 21:00 UTC (= 06:00 KST)
    └─▶ wakeup.yml 실행
            └─ wakeup 테이블 created_at 갱신 (service_role 키 사용)
```

## 5. 로컬 개발

```bash
# .env.example을 복사해서 .env 생성 (git에 커밋되지 않음)
cp .env.example .env
# 편집기로 .env를 열어 실제 값 입력
```

로컬에서 `index.html`을 직접 열면 `__SUPABASE_URL__` 플레이스홀더가 치환되지 않아 설정 안내 메시지가 표시됩니다. 로컬 확인이 필요한 경우 아래 명령어로 플레이스홀더를 직접 치환하세요:

```bash
# macOS / Linux
source .env
cp index.html index.local.html
sed -i "s|__SUPABASE_URL__|${SUPABASE_URL}|g" index.local.html
sed -i "s|__SUPABASE_ANON_KEY__|${SUPABASE_ANON_KEY}|g" index.local.html
open index.local.html   # 확인 후 삭제
```

## 오래된 run 정리 (선택)
`cleanup.yml`은 매일 **09:00 KST**(`cron: "0 0 * * *"`)에 워크플로 run 기록을 정리해, **최신 10개만 남기고** 나머지를 삭제합니다.
- run 삭제에는 `actions: write` 권한이 필요해서 워크플로 안에 `permissions: actions: write`를 넣어뒀습니다(추가 Secret 불필요, 기본 `GITHUB_TOKEN` 사용).
- 보관 개수는 `KEEP`, 특정 워크플로만 정리하려면 `WORKFLOW: wakeup.yml` 처럼 지정하세요(비우면 저장소 전체 대상).

## 알아두면 좋은 점
- **cron 지연:** GitHub 무료 러너의 예약 실행은 부하에 따라 수 분~수십 분 늦을 수 있습니다. 정시 보장이 필요하면 별도 스케줄러(예: Supabase `pg_cron`, 외부 cron)를 고려하세요.
- **60일 규칙:** 저장소에 60일간 커밋·활동이 없으면 GitHub가 예약 워크플로를 자동 비활성화합니다(이메일로 재활성화 링크 제공).
- **id 변경:** 다른 행을 갱신하려면 `wakeup.yml`의 `?id=eq.1` 과 시계의 조회 로직을 맞춰 바꾸세요.
