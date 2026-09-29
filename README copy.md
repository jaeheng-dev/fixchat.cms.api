# statkit.cms.api

## 프로젝트 소개

**statkit.cms.api**는 별도로 운영되는 일반 홈페이지(고객용 프론트엔드)에 노출할 콘텐츠를 관리하는 **헤드리스 CMS 관리자(Admin) 서비스**입니다.

- **화면 성격**: 방문자용 페이지가 아닌 관리자 전용 백오피스입니다.
- **핵심 역할**: 블로그, 상품, 포트폴리오, 주문, 견적, 회원 등 콘텐츠를 CRUD하고, 콘텐츠 유형별로 `/api/v1/{resource}` REST API 노출 여부를 관리 페이지에서 토글 버튼으로 즉시 제어합니다 (`statkit_content_api_configs` 테이블 기반, 코드 배포 없이 on/off).
- **데이터 흐름**: 관리자가 이 서비스에서 콘텐츠를 등록·수정 → Supabase DB 반영 → 활성화된 리소스는 동적 REST API로 노출 → 별도의 일반 홈페이지가 해당 API를 호출해 화면에 표시합니다.
- **API 문서**: 활성화된 콘텐츠 기준으로 OpenAPI 스펙(`/api/openapi.json`)이 동적으로 생성되어 항상 실제 DB 상태와 일치합니다.

---

## 기술 스택

| 항목 | 내용 |
|------|------|
| 프레임워크 | Next.js 14 (App Router) |
| 전역 상태 | Recoil |
| 데이터 패칭 | @tanstack/react-query |
| HTTP 클라이언트 | Axios |
| UI 컴포넌트 | shadcn/ui |
| 리치 텍스트 에디터 | Tiptap (블로그/포트폴리오 본문 작성) |
| 폼 관리 | react-hook-form + Zod |
| 인증 | @supabase/ssr |
| API 스펙 · 문서 | zod-to-openapi + swagger-ui-react (`/api-docs`) |
| 토스트 알림 | Sonner |
| 개발 포트 | 3001 |

---

## 프로젝트 구조

콘텐츠 관리(백오피스 화면)와 REST API 노출(`/api/v1`)이 한 코드베이스 안에 공존하는 구조입니다.

```
statkit.cms.api/
├── docs/                          # PRD, ROADMAP
├── supabase/
│   ├── migrations/                # 스키마 정의 (SQL Editor에 수동 실행)
│   └── backups/                   # 테이블 데이터 백업
├── src/
│   ├── middleware.ts               # 라우트 보호 (현재 mock-auth 쿠키 기반, Supabase 세션 전환 예정)
│   ├── app/
│   │   ├── (admin)/                    # 관리자 전용 콘텐츠 관리 화면
│   │   │   ├── blog/                   # 블로그(posts) CRUD
│   │   │   ├── products/               # 상품 CRUD
│   │   │   ├── portfolio/              # 포트폴리오 CRUD
│   │   │   ├── orders/                 # 주문 관리
│   │   │   ├── quotes/                 # 견적 문의 관리
│   │   │   ├── users/                  # 회원(profiles) 관리
│   │   │   ├── settings/               # 리소스별 REST API 노출 on/off
│   │   │   └── profile/                # 관리자 본인 계정
│   │   ├── api/
│   │   │   ├── v1/[resource]/          # 활성화된 콘텐츠를 노출하는 동적 목록 API
│   │   │   ├── v1/[resource]/[identifier]/  # 동적 단건 조회 API
│   │   │   ├── v1/health/              # 헬스체크
│   │   │   ├── openapi.json/           # 활성화된 config 기반 OpenAPI 스펙 (동적 생성)
│   │   │   └── auth/me/                # 현재 로그인 사용자 조회
│   │   ├── api-docs/                   # Swagger UI (공개 API 문서)
│   │   ├── auth/callback/              # OAuth 콜백
│   │   ├── sign/in/, sign/up/          # 로그인 · 회원가입
│   │   └── layout.tsx, page.tsx
│   ├── components/
│   │   ├── admin/
│   │   │   ├── AdminShell/, Sidebar/       # 관리자 레이아웃 · 내비게이션
│   │   │   ├── ApiConfigCard/              # 리소스별 API 노출 토글 카드 (Settings 페이지)
│   │   │   ├── PostForm/, ProductForm/, PortfolioForm/  # 콘텐츠 작성 폼
│   │   │   └── RichEditor/                 # Tiptap 기반 에디터
│   │   ├── auth/AuthProvider/          # 인증 상태 초기화 (onAuthStateChange)
│   │   ├── swagger/SwaggerUI/          # /api-docs 렌더링
│   │   └── ui/                         # shadcn/ui 컴포넌트
│   ├── hooks/
│   │   ├── useAuth.ts              # 인증 상태 + signOut
│   │   └── useUser.ts
│   ├── libs/
│   │   ├── supabase/
│   │   │   ├── client.ts, server.ts, admin.ts  # 클라이언트/서버/서비스롤 클라이언트
│   │   │   ├── middleware.ts       # updateSession() 헬퍼
│   │   │   └── queries/            # 모든 Supabase 쿼리 (posts, products, portfolios, content-api-configs …)
│   │   └── openapi/
│   │       ├── dynamic-spec.ts     # 활성화된 config로 OpenAPI 스펙 생성
│   │       └── schemas/            # 리소스별 타입 스펙
│   └── store/
│       ├── atoms/                  # authAtom, userAtom, uiAtom
│       └── selectors/authSelector.ts  # isAuthenticated, isAdmin
└── vercel.json
```

---

## 환경변수 설정

프로젝트 루트에 `.env.local`을 생성합니다. (`.env.example` 참고)

```env
# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=sb_publishable_...
SUPABASE_SERVICE_ROLE_KEY=sb_secret_...   # 서버 전용, 절대 클라이언트 노출 금지

# OAuth (Supabase 대시보드에서 설정 시 사용)
GOOGLE_OAUTH_ID=
GOOGLE_OAUTH_SECRET=
KAKAO_CLIENT_ID=
KAKAO_CLIENT_SECRET=

# 공개 REST API(/api/v1/*) 인증 키 — 외부 프론트엔드가 X-API-Key 헤더로 전달
INTERNAL_API_KEY=
```

Supabase 프로젝트 대시보드 → **Settings → API Keys** 에서 값을 확인할 수 있습니다. `INTERNAL_API_KEY`가 없으면 `/api/v1/*` 요청은 500(서버 설정 오류)을 반환합니다.

---

## 설치 및 실행

```bash
# 의존성 설치
yarn install

# 개발 서버 (포트 3001)
yarn dev

# 프로덕션 빌드
yarn build
yarn start

# 린트 / 포맷
yarn lint
yarn pretty
```

---

## Supabase 스키마 설정

Supabase 스키마는 **대시보드 SQL Editor에서 직접 실행**합니다 (로컬 CLI 마이그레이션 미사용).

### profiles 테이블

```sql
CREATE TABLE public.profiles (
  id UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
  role TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user', 'admin')),
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- 회원가입 시 자동 생성 트리거
CREATE OR REPLACE FUNCTION public.handle_new_user()
RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO public.profiles (id, email, full_name, avatar_url)
  VALUES (NEW.id, NEW.email, NEW.raw_user_meta_data->>'full_name', NEW.raw_user_meta_data->>'avatar_url');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;

CREATE TRIGGER on_auth_user_created
  AFTER INSERT ON auth.users
  FOR EACH ROW EXECUTE FUNCTION public.handle_new_user();
```

### ⚠️ RLS (Row Level Security) 필수 활성화

`profiles` 테이블 생성 후 반드시 RLS를 활성화해야 합니다. **RLS 없이는 `anon` 키를 가진 모든 클라이언트가 전체 프로필을 조회·수정할 수 있으며, 일반 사용자가 본인의 `role`을 직접 변경할 수 있습니다.**

```sql
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;

-- 본인 프로필만 조회 가능
CREATE POLICY "profiles_select_own" ON public.profiles
  FOR SELECT USING (auth.uid() = id);

-- 본인 프로필 수정 가능 (role 컬럼은 변경 불가)
CREATE POLICY "profiles_update_own" ON public.profiles
  FOR UPDATE USING (auth.uid() = id)
  WITH CHECK (
    auth.uid() = id
    AND role = (SELECT role FROM public.profiles WHERE id = auth.uid())
  );
```

### 콘텐츠 테이블

`profiles` 외에 실제 콘텐츠를 담는 테이블은 `supabase/migrations/`에 정의되어 있습니다. 전체 DDL은 마이그레이션 파일을 기준으로 확인하세요 — README에는 최신성이 깨지기 쉬운 SQL 전문 대신 요약만 둡니다.

| 테이블 | 관리 페이지 | 비고 |
|--------|-------------|------|
| `statkit_posts` | `/blog` | 블로그 글 |
| `statkit_products` | `/products` | 상품 |
| `statkit_portfolios` | `/portfolio` | 포트폴리오 (관리자 CRUD 기준) |
| `orders` | `/orders` | 주문 |
| `quote_submissions` | `/quotes` | 견적 문의 |
| `statkit_content_api_configs` | `/settings` | 리소스별 REST API 노출 설정 (`resource_slug`, `table_name`, `is_enabled` 등) |

`statkit_` 접두사가 붙은 테이블은 `002_statkit_prefix_tables.sql`에서 도입된 명명 규칙이며, 신규 콘텐츠 테이블 추가 시에도 이 접두사를 따릅니다.

> ⚠️ **알려진 스키마 드리프트**: `002_statkit_prefix_tables.sql`의 `statkit_content_api_configs` 시드 데이터는 `portfolio` 리소스의 `table_name`을 `portfolios`(접두사 없음)로 등록하지만, 관리자 CRUD(`portfolios.admin.ts`)는 `statkit_portfolios`를 사용합니다. 즉 관리자가 쓰는 테이블과 공개 API가 읽을 테이블이 다릅니다. 현재 `portfolio` 리소스의 `is_enabled`가 `false`라 당장 영향은 없지만, 이 API를 활성화하기 전에 `table_name`을 `statkit_portfolios`로 맞추는 수정이 필요합니다.

---

## 콘텐츠 관리 및 동적 REST API

이 서비스의 핵심은 **관리자가 콘텐츠를 만들면, 코드 변경 없이 REST API로 노출할 수 있다**는 점입니다.

1. 관리자가 `/blog`, `/products`, `/portfolio` 등에서 콘텐츠를 작성·수정합니다.
2. `/settings` 페이지의 `ApiConfigCard`에서 리소스별로 API 노출 여부(`is_enabled`)를 토글합니다 — 이 값은 `statkit_content_api_configs` 테이블 한 행으로 관리되며, 새 콘텐츠 유형이 추가돼도 라우트 코드를 새로 작성할 필요가 없습니다.
3. 활성화된 리소스는 `GET /api/v1/{resource}`(목록), `GET /api/v1/{resource}/{identifier}`(단건)로 조회 가능해집니다. 두 라우트 모두 `src/app/api/v1/[resource]/`의 단일 동적 핸들러가 `statkit_content_api_configs` 설정을 읽어 응답하며, 호출 시 `X-API-Key` 헤더 인증이 필요합니다(자세한 내용은 [인증 흐름](#인증-흐름) 참고).
4. `GET /api/openapi.json`이 현재 활성화된 리소스만 반영한 OpenAPI 스펙을 매 요청마다 동적으로 생성하고, `/api-docs`(Swagger UI)가 이를 사람이 읽을 수 있는 문서로 보여줍니다.
5. `GET /api/v1/health`로 API 서버 상태를 확인할 수 있습니다.

즉, 일반 홈페이지(고객용 프론트엔드) 개발자는 이 서비스의 `/api/v1/*` 엔드포인트만 호출하면 되고, 관리자는 어떤 콘텐츠를 얼마나 노출할지 화면에서 직접 통제합니다.

---

## 인증 흐름

관리자 화면(백오피스)과 공개 REST API는 인증 방식이 다릅니다.

### 관리자 화면 (`(admin)` 라우트 그룹)

- `/sign/in`, `/sign/up` — 공개
- `/auth/callback` — OAuth 콜백
- `/users`, `/orders`, `/quotes`, `/blog`, `/products`, `/portfolio`, `/profile`, `/settings` — 보호 (비인증 시 `/sign/in` 리다이렉트)
- `/api-docs`(Swagger UI) — 로그인된 관리자만 접근 가능
- 로그인 상태에서 `/sign/in` 접근 시 `/users`로 리다이렉트

> ⚠️ **현재 `src/middleware.ts`는 `mock-auth` 쿠키로 인증 여부를 판단하는 개발용 인증**입니다 (`src/libs/supabase/middleware.ts`의 `updateSession()`은 정의만 되어 있고 아직 미들웨어에 연결되지 않았습니다). 실서비스 전환 시 Supabase 세션 쿠키 기반 검증으로 교체가 필요합니다.

`AuthProvider`(`src/components/auth/AuthProvider/index.tsx`)는 클라이언트에서 `onAuthStateChange`를 구독하여 Recoil atoms(`sessionAtom`, `authUserAtom`, `userProfileAtom`)를 동기화합니다.

### 공개 REST API (`/api/v1/*`, `/api/openapi.json`)

- 관리자 세션이 아닌 **`X-API-Key` 헤더**로 인증합니다 (`src/libs/openapi/auth.ts`의 `validateApiKey()` → `INTERNAL_API_KEY` 환경변수와 비교).
- CORS는 모든 origin에 허용되어 있어(`Access-Control-Allow-Origin: *`), 일반 홈페이지 등 외부 프론트엔드가 브라우저에서 직접 호출할 수 있습니다.
- 키가 없거나 일치하지 않으면 401을 반환합니다.
