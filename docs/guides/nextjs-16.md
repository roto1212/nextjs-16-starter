# Next.js 16.4.6 개발 지침

이 문서는 Claude Code에서 Next.js 16.4.6 프로젝트를 개발할 때 따라야 할 핵심 규칙과 가이드라인을 제공합니다.

## 🚀 필수 규칙 (엄격 준수)

### App Router 아키텍처

```typescript
// ✅ 올바른 방법: App Router 사용
app/
├── layout.tsx          // 루트 레이아웃
├── page.tsx           // 메인 페이지
├── loading.tsx        // 로딩 UI
├── error.tsx          // 에러 UI
├── not-found.tsx      // 404 페이지
└── dashboard/
    ├── layout.tsx     // 대시보드 레이아웃
    └── page.tsx       // 대시보드 페이지

// ❌ 금지: Pages Router 사용
pages/
├── index.tsx
└── dashboard.tsx
```

### Server Components 우선 설계

```typescript
// 🚀 필수: 기본적으로 모든 컴포넌트는 Server Components
export default async function UserDashboard() {
  // 서버에서 데이터 가져오기
  const user = await getUser()

  return (
    <div>
      <h1>{user.name}님의 대시보드</h1>
      {/* 클라이언트 컴포넌트가 필요한 경우에만 분리 */}
      <InteractiveChart data={user.analytics} />
    </div>
  )
}

// ✅ 클라이언트 컴포넌트는 최소한으로 사용
'use client'

import { useState } from 'react'

export function InteractiveChart({ data }: { data: Analytics[] }) {
  const [selectedRange, setSelectedRange] = useState('week')
  // 상호작용 로직만 클라이언트에서 처리
  return <Chart data={data} range={selectedRange} />
}
```

### 🔄 async request APIs 처리 (16에서 동기 접근 완전 제거)

```typescript
// 🔄 Next.js 16.4.6 방식 (npx next typegen 으로 PageProps/LayoutProps 헬퍼 타입 생성 가능)
import { cookies, headers } from 'next/headers'

export default async function Page({
  params,
  searchParams
}: {
  params: Promise<{ id: string }>
  searchParams: Promise<{ [key: string]: string | string[] | undefined }>
}) {
  // 🚀 필수: async request APIs 올바른 처리
  const { id } = await params
  const query = await searchParams
  const cookieStore = await cookies()
  const headersList = await headers()

  const user = await getUser(id)

  return <UserProfile user={user} />
}

// ❌ 금지: 동기식 접근 (16에서 완전 제거, 15의 임시 호환 불가)
export default function Page({ params }: { params: { id: string } }) {
  const user = getUser(params.id) // 에러 발생
  return <UserProfile user={user} />
}
```

### Typed Routes 활용

```typescript
// 🚀 필수: Typed Routes로 타입 안전성 보장
import Link from 'next/link'

// next.config.ts에서 typedRoutes: true 설정 필요 (15.5부터 stable, 16에서는 최상위 옵션)
export function Navigation() {
  return (
    <nav>
      {/* ✅ 타입 안전한 링크 */}
      <Link href="/dashboard/users/123">사용자 상세</Link>
      <Link href={{
        pathname: '/products/[id]',
        params: { id: 'abc' }
      }}>제품 상세</Link>

      {/* ❌ 컴파일 에러: 존재하지 않는 경로 */}
      <Link href="/nonexistent-route">잘못된 링크</Link>
    </nav>
  )
}
```

## ✅ 권장 사항 (성능 최적화)

### Streaming과 Suspense 활용

```typescript
import { Suspense } from 'react'

export default function DashboardPage() {
  return (
    <div>
      <h1>대시보드</h1>

      {/* ✅ 빠른 컨텐츠는 즉시 렌더링 */}
      <QuickStats />

      {/* ✅ 느린 컨텐츠는 Suspense로 감싸기 */}
      <Suspense fallback={<SkeletonChart />}>
        <SlowChart />
      </Suspense>

      <Suspense fallback={<SkeletonTable />}>
        <SlowDataTable />
      </Suspense>
    </div>
  )
}

async function SlowChart() {
  // 무거운 데이터 처리
  await new Promise(resolve => setTimeout(resolve, 2000))
  const data = await getComplexAnalytics()

  return <Chart data={data} />
}
```

### after() API 활용

```typescript
import { after } from 'next/server'

export async function POST(request: Request) {
  const body = await request.json()

  // 즉시 응답 반환
  const result = await processUserData(body)

  // 🔄 비블로킹 작업은 after()로 처리
  after(async () => {
    await sendAnalytics(result)
    await updateCache(result.id)
    await sendNotification(result.userId)
  })

  return Response.json({ success: true, id: result.id })
}
```

### 캐싱 전략 (Cache Components)

```typescript
// next.config.ts
const nextConfig: NextConfig = {
  cacheComponents: true, // 🔄 16: 명시적 opt-in 캐싱 (기존 experimental.ppr / dynamicIO 대체)
}
```

```typescript
// ✅ 16의 기본 동작: 모든 코드는 요청 시점에 동적 실행.
//    캐시하려면 'use cache' 지시어를 명시한다 (opt-in).
import { cacheLife, cacheTag } from 'next/cache'

export async function getProductData(id: string) {
  'use cache'
  cacheLife('hours') // 캐시 수명 프로필
  cacheTag(`product-${id}`, 'products') // 태그 기반 무효화

  const res = await fetch(`https://api.example.com/products/${id}`)
  return res.json()
}
```

```typescript
// 캐시 무효화 (Server Action / Route Handler)
import { revalidateTag, updateTag, refresh } from 'next/cache'

export async function updateProduct(id: string, data: ProductData) {
  'use server'
  await updateDatabase(id, data)

  // 🔄 16: revalidateTag는 cacheLife 프로필(2번째 인자) 필수 — stale-while-revalidate
  revalidateTag('products', 'max')

  // 🔄 16 신규: updateTag — Server Action 전용, 즉시 만료 후 같은 요청에서 최신 데이터 확인 (read-your-writes)
  updateTag(`product-${id}`)
}

// refresh(): 캐시되지 않은 동적 데이터만 새로고침 (Server Action 전용)
```

> 15의 `fetch(..., { next: { revalidate, tags } })`는 `cacheComponents` 미사용 시에만 유효합니다.
> 신규 코드는 `'use cache'` + `cacheLife`/`cacheTag`를 사용하세요.

### Turbopack (16부터 기본 번들러)

```typescript
// next.config.ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  // ✅ Turbopack은 dev/build 모두 기본값 → `--turbopack` 플래그 불필요
  //    (webpack이 꼭 필요하면 `next build --webpack`)
  // 🔄 turbopack 옵션은 experimental.turbo 가 아닌 최상위 `turbopack`
  turbopack: {
    // rules, resolveAlias 등
  },
  // 🔄 dev 파일시스템 캐시 (선택, 대형 프로젝트 재시작 가속)
  // experimental: { turbopackFileSystemCacheForDev: true },
  experimental: {
    optimizePackageImports: ['lucide-react'],
  },
  // 🔄 React Compiler (stable, 자동 메모이제이션) — 필요 시 활성화
  // 설치: pnpm add -D babel-plugin-react-compiler
  // reactCompiler: true,
}

export default nextConfig
```

## ⚠️ Breaking Changes 대응

### React 19 호환성

```typescript
// ⚠️ React 19에서 변경된 사항들

// ✅ 새로운 방식: useFormStatus 훅
'use client'

import { useFormStatus } from 'react-dom'

function SubmitButton() {
  const { pending } = useFormStatus()

  return (
    <button type="submit" disabled={pending}>
      {pending ? '제출 중...' : '제출'}
    </button>
  )
}

// ✅ Server Actions와 form 통합
export async function createUser(formData: FormData) {
  'use server'

  const name = formData.get('name') as string
  const email = formData.get('email') as string

  await saveUser({ name, email })
  redirect('/users')
}

export default function UserForm() {
  return (
    <form action={createUser}>
      <input name="name" required />
      <input name="email" type="email" required />
      <SubmitButton />
    </form>
  )
}
```

### middleware → proxy (파일/함수명 변경)

```typescript
// ❌ 16에서 deprecated: middleware.ts + export function middleware
// ✅ proxy.ts + export function proxy (Node.js runtime 고정, Edge 불가)
// 마이그레이션: npx @next/codemod@canary middleware-to-proxy .
import { NextRequest, NextResponse } from 'next/server'

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)'],
}

export function proxy(request: NextRequest) {
  const token = request.cookies.get('auth-token')?.value

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return NextResponse.next()
}
```

> ⚠️ proxy는 네트워크 경계의 가벼운 리다이렉트/헤더 처리 용도입니다.
> 인가(권한) 검증은 반드시 데이터 접근 계층(Server Component / Server Action / Route Handler)에서 다시 수행하세요.
> (Next.js 16.x 보안 이슈 교훈: 미들웨어 단일 검증에 의존하지 않기)

### 기타 16 Breaking Changes

| 항목                            | 16 변경 사항                                                                                                                 |
| ------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| 요구 사양                       | Node.js 20.9+, TypeScript 5.1+                                                                                               |
| `next lint`                     | 제거 → `eslint .` 직접 실행 (`eslint-config-next` flat config)                                                               |
| `next/image`                    | `images.minimumCacheTTL` 기본 4시간, 쿼리스트링 있는 로컬 이미지는 `images.localPatterns` 설정 필요, `qualities` 기본 `[75]` |
| Parallel Routes                 | 모든 슬롯에 `default.tsx` 명시 필수 (없으면 빌드 실패)                                                                       |
| AMP, `serverRuntimeConfig`      | 제거 → 환경변수 사용                                                                                                         |
| `experimental.ppr`, `dynamicIO` | `cacheComponents`로 통합                                                                                                     |
| React                           | 19.2 (View Transitions, `useEffectEvent`, Activity)                                                                          |

### unauthorized/forbidden API (experimental)

```typescript
// app/api/admin/route.ts
import { unauthorized, forbidden } from 'next/navigation'
// ⚠️ next.config.ts에서 experimental.authInterrupts: true 필요

export async function GET(request: Request) {
  const session = await getSession(request)

  // 🔄 새로운 unauthorized 함수
  if (!session) {
    unauthorized()
  }

  // 🔄 새로운 forbidden 함수
  if (!session.user.isAdmin) {
    forbidden()
  }

  const data = await getAdminData()
  return Response.json(data)
}
```

## 고급 라우팅 패턴

### Route Groups 고급 패턴

```typescript
// ✅ Route Groups로 레이아웃 분리
app/
├── (marketing)/
│   ├── layout.tsx     // 마케팅 레이아웃
│   ├── page.tsx       // 홈페이지
│   └── about/
│       └── page.tsx   // 소개 페이지
├── (dashboard)/
│   ├── layout.tsx     // 대시보드 레이아웃
│   └── analytics/
│       └── page.tsx   // 분석 페이지
└── (auth)/
    ├── login/
    │   └── page.tsx
    └── register/
        └── page.tsx

// (marketing)/layout.tsx
export default function MarketingLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    <div className="marketing-layout">
      <MarketingHeader />
      {children}
      <MarketingFooter />
    </div>
  )
}
```

### Parallel Routes 활용

```typescript
// ✅ Parallel Routes로 동시 렌더링
app/
├── dashboard/
│   ├── layout.tsx
│   ├── page.tsx
│   ├── @analytics/
│   │   └── page.tsx
│   └── @notifications/
│       └── page.tsx

// dashboard/layout.tsx
export default function DashboardLayout({
  children,
  analytics,
  notifications,
}: {
  children: React.ReactNode
  analytics: React.ReactNode
  notifications: React.ReactNode
}) {
  return (
    <div className="dashboard-grid">
      <main>{children}</main>
      <aside className="analytics-panel">
        <Suspense fallback={<AnalyticsSkeleton />}>
          {analytics}
        </Suspense>
      </aside>
      <div className="notifications-panel">
        <Suspense fallback={<NotificationsSkeleton />}>
          {notifications}
        </Suspense>
      </div>
    </div>
  )
}
```

### Intercepting Routes

```typescript
// ✅ Intercepting Routes로 모달 구현
app/
├── gallery/
│   ├── page.tsx
│   └── [id]/
│       └── page.tsx    // 전체 페이지 보기
└── @modal/
    └── (.)gallery/
        └── [id]/
            └── page.tsx // 모달 보기

// @modal/(.)gallery/[id]/page.tsx
import { Modal } from '@/components/modal'

export default async function PhotoModal({
  params
}: {
  params: Promise<{ id: string }>
}) {
  const { id } = await params
  const photo = await getPhoto(id)

  return (
    <Modal>
      <img src={photo.url} alt={photo.title} />
    </Modal>
  )
}
```

## ❌ 금지 사항

### Pages Router 사용 금지

```typescript
// ❌ 절대 금지: Pages Router 패턴
pages/
├── _app.tsx
├── _document.tsx
├── index.tsx
└── api/
    └── users.ts

// ❌ 금지: getServerSideProps, getStaticProps 사용
export async function getServerSideProps() {
  // 이 방식은 사용하지 마세요
}
```

### 안티패턴 방지

```typescript
// ❌ 금지: 불필요한 'use client' 사용
'use client'

export default function SimpleComponent({ title }: { title: string }) {
  // 상태나 이벤트 핸들러가 없는데 'use client' 사용
  return <h1>{title}</h1>
}

// ✅ 올바른 방법: Server Component로 유지
export default function SimpleComponent({ title }: { title: string }) {
  return <h1>{title}</h1>
}

// ❌ 금지: 클라이언트에서 서버 함수 직접 호출
'use client'

import { getUser } from '@/lib/database' // 서버 전용 함수

export function UserProfile() {
  const user = getUser() // 에러 발생
  return <div>{user.name}</div>
}

// ✅ 올바른 방법: 서버에서 데이터 전달
export default async function UserPage() {
  const user = await getUser()
  return <UserProfile user={user} />
}

function UserProfile({ user }: { user: User }) {
  return <div>{user.name}</div>
}
```

## 코드 품질 체크리스트

개발 완료 후 다음 명령어들을 반드시 실행하세요:

```bash
# 🚀 필수: 타입 체크
pnpm typecheck

# 🚀 필수: 린트 검사
pnpm lint

# ✅ 권장: 포맷 검사
pnpm format:check

# 🚀 필수: 통합 검사
pnpm check-all

# 🚀 필수: 빌드 테스트
pnpm build
```

이 지침을 따라 Next.js 16.4.6의 모든 기능을 최대한 활용하여 현대적이고 성능 최적화된 애플리케이션을 개발하세요.
