## notFound
- path: https://github.com/vercel/next.js/blob/e1b23eb16af257cce80ab7c6893617ff34b65bd9/packages/next/src/client/components/not-found.ts

* In a Server Component, this will insert a `<meta name="robots" content="noindex" />` meta tag and set the status code to 404.
* In a Route Handler or Server Action, it will serve a 404 to the caller.

```tsx
export function notFound(): never {
```

- 반환 타입을 `never`로 선언.  
    → 정상적으로 값을 리턴하는 경우가 없고, 함수가 항상 예외를 throw하기 때문.

```tsx
const error = new Error(DIGEST) as HTTPAccessFallbackError
```
- 새로운 Error 객체를 만들면서 로 메시지 - DIGEST
- `Error` 객체를 `HTTPAccessFallbackError` 타입이라고 강제 캐스팅

이렇게 throw된 에러는 Next.js의 에러 처리 시스템에 의해 잡힘
    - 서버 컴포넌트에서는 404 상태 코드와 함께 `not-found.js` 페이지를 렌더링
    - 라우트 핸들러에서는 404 응답을 반환

이 에러 바운더리에 의해 
https://github.com/vercel/next.js/blob/0b136b273e6e7f8727d591209111b1d949e62945/packages/next/src/client/components/http-access-fallback/error-boundary.tsx

Notfound 컴포넌트가 렌더링
https://github.com/vercel/next.js/blob/997105d27ebc7bfe01b7e907cd659e5e056e637c/packages/next/src/client/components/not-found-error.tsx

```tsx
import { HTTPAccessErrorFallback } from './http-access-fallback/error-fallback'

export default function NotFound() {
  return (
    <HTTPAccessErrorFallback
      status={404}
      message="This page could not be found."
    />
  )
}
```
## redirect
- path: https://github.com/vercel/next.js/blob/9118a0b609a7b701d4fd4bd7846dd939c5076428/packages/next/src/client/components/redirect.ts
- 이 에러는 나중에 `RedirectBoundary` 컴포넌트에서 캐치되어 실제 리다이렉션을 수행합니다. 리다이렉션 처리는 서버 컴포넌트, 라우트 핸들러, 서버 액션에서 각각 다르게 동작합니다:
	- 서버 컴포넌트: insert a meta tag to redirect the user to the target page.
	- 라우트 핸들러: it will serve a 307/303 to the caller.
	- 서버 액션: type defaults to 'push' and 'replace' elsewhere.

이러한 구현 방식은 Next.js의 다양한 컨텍스트에서 리다이렉션을 일관되게 처리할 수 있게 해줍니다.


```tsx
export function redirect(
/** The URL to redirect to */
	url: string,
	type?: RedirectType
): never {
	type ??= actionAsyncStorage?.getStore()?.isAction
		? RedirectType.push
		: RedirectType.replace
	
	throw getRedirectError(url, type, RedirectStatusCode.TemporaryRedirect)
}
```
- type
	- `RedirectType.push`: 브라우저 히스토리에 새로운 항목을 추가 (뒤로 가기 가능)
	- `RedirectType.replace`: 현재 히스토리 항목을 새로운 URL로 대체 (뒤로 가기 불가)
	- https://nextjs.org/docs/app/api-reference/functions/redirect#parameters
	- Server Action과 그 외의 경우에서 리다이렉트 동작을 다르게 설계한 이유는 각 상황의 사용자 경험(UX)과 일반적인 웹 패턴을 고려했기 때문입니다:
		1. **Server Action에서 `push` 사용 (`RedirectType.push`)**:
		    - Server Action은 주로 폼 제출, 데이터 생성/수정과 같은 사용자의 의도적인 액션에서 사용됩니다
		    - 예시:
```tsx
// 새 게시글 작성 후 리다이렉트
async function createPost(formData: FormData) {
  'use server'
  const post = await db.posts.create(...)
  redirect(`/posts/${post.id}`) // push 사용
}
```
			- 이런 경우 사용자가 이전 페이지(예: 폼 페이지)로 돌아가고 싶을 수 있습니다
		    - `push`를 사용하면 브라우저의 뒤로 가기를 통해 이전 상태로 돌아갈 수 있습니다


		2. **그 외의 경우에서 `replace` 사용 (`RedirectType.replace`)**:
		    - Server Component나 Route Handler에서의 리다이렉트는 주로 인증, 권한 검사, 페이지 이동 로직에서 사용됩니다
		    - 예시:
```tsx
        // 인증되지 않은 사용자 리다이렉트
        if (!isAuthenticated) {
          redirect('/login') // replace 사용
        }
```

    - 이런 경우 이전 페이지로 돌아가는 것이 의미가 없거나 보안상 바람직하지 않을 수 있습니다
    - `replace`를 사용하면 현재 페이지를 새 페이지로 완전히 대체하므로, 불필요한 히스토리 스택 쌓임을 방지할 수 있습니다

이러한 설계는 Next.js의 문서에서도 확인할 수 있습니다:
이 로직은 Server Action에서는 사용자가 뒤로 가기를 할 수 있도록 하고, 그 외의 경우에는 현재 페이지를 새로운 페이지로 대체하는 방식으로 동작하도록 설계되어 있습니다.


- getRedirectError를 throw
```tsx
export enum RedirectStatusCode {
	SeeOther = 303,
	TemporaryRedirect = 307,
	PermanentRedirect = 308,
}
export function getRedirectError(
	url: string,
	type: RedirectType,
	statusCode: RedirectStatusCode = RedirectStatusCode.TemporaryRedirect
): RedirectError {
	const error = new Error(REDIRECT_ERROR_CODE) as RedirectError
	error.digest = `${REDIRECT_ERROR_CODE};${type};${url};${statusCode};`
	return error
}
```

### getRedirectError를 캐치하는 `RedirectErrorBoundary`
```ts
export class RedirectErrorBoundary extends React.Component<
  RedirectBoundaryProps,
  { redirect: string | null; redirectType: RedirectType | null }
> {
  constructor(props: RedirectBoundaryProps) {
    super(props)
    this.state = { redirect: null, redirectType: null }
  }

  static getDerivedStateFromError(error: any) {
    if (isRedirectError(error)) {
      const url = getURLFromRedirectError(error)
      const redirectType = getRedirectTypeFromError(error)
      return { redirect: url, redirectType }
    }
    // Re-throw if error is not for redirect
    throw error
  }

  // Explicit type is needed to avoid the generated `.d.ts` having a wide return type that could be specific to the `@types/react` version.
  render(): React.ReactNode {
    const { redirect, redirectType } = this.state
    // - 리다이렉트 정보가 있으면 `HandleRedirect` 컴포넌트 렌더링
	//- 없으면 자식 컴포넌트 렌더링
    if (redirect !== null && redirectType !== null) {
      return (
        <HandleRedirect
          redirect={redirect}
          redirectType={redirectType}
          reset={() => this.setState({ redirect: null })}
        />
      )
    }

    return this.props.children
  }
}

function HandleRedirect({
  redirect,
  reset,
  redirectType,
}: {
  redirect: string
  redirectType: RedirectType
  reset: () => void
}) {
  const router = useRouter()

  useEffect(() => {
    React.startTransition(() => {
      if (redirectType === RedirectType.push) {
        router.push(redirect, {})
      } else {
        router.replace(redirect, {})
      }
      reset()
    })
  }, [redirect, redirectType, reset, router])

  return null
}

```
RedirectErrorBoundary가 감싸져있는 layout
https://github.com/vercel/next.js/blob/35864749113a10d7943a2df9d046c218e7ade023/packages/next/src/client/components/layout-router.tsx

## permanentRedirect

- path: https://github.com/vercel/next.js/blob/9118a0b609a7b701d4fd4bd7846dd939c5076428/packages/next/src/client/components/redirect.ts#L61
- 똑같이 `getRedirectError`를 throw하는데 `RedirectStatusCode.PermanentRedirect`를 전달
	* In a Server Component, this will insert a meta tag to redirect the user to the target page.
	* In a Route Handler or Server Action, it will serve a 308/303 to the caller.

```ts
export function permanentRedirect(
  /** The URL to redirect to */
  url: string,
  type: RedirectType = RedirectType.replace
): never {
  throw getRedirectError(url, type, RedirectStatusCode.PermanentRedirect)
}
```


>  [Why does `redirect` use 307 and 308?](https://nextjs.org/docs/app/api-reference/functions/redirect#why-does-redirect-use-307-and-308)> 

HTTP에는 리다이렉트를 나타내는 여러 상태 코드가 있어:

- **301**: Permanent Redirect (영구 이동)
    
- **302**: Temporary Redirect (임시 이동)
    

**문제는 302**야.  
302를 사용할 때, **브라우저들이 원래 요청 방식(method)을 무시하고 POST 요청을 GET 요청으로 바꿔버리는** 문제가 있었어.

예를 들어:

- `/users`로 `POST` 요청(새 사용자 생성)을 했어.
    
- 서버가 302로 `/people`로 리다이렉트했어.
    
- 브라우저는 **POST가 아니라 GET으로** `/people`을 요청해버림. ❌
    

→ **문제가 생기는 이유**: 사용자 생성은 POST로 해야지, GET으로 하면 데이터가 안 들어가잖아.

---

### 그래서 등장한 해결책: 307과 308

새로운 리다이렉트 상태 코드가 등장했어:

|코드|의미|요청 메소드 (GET, POST 등) 유지 여부|
|---|---|---|
|302|임시 이동 (하지만 method가 **변경**될 수 있음)|❌ 유지 안 됨|
|307|임시 이동 (**method 유지**)|✅ 유지됨|
|301|영구 이동 (method가 **변경**될 수 있음)|❌ 유지 안 됨|
|308|영구 이동 (**method 유지**)|✅ 유지됨|

- **307**: Temporary redirect인데, POST/GET 등 **요청 메소드가 그대로 유지**돼.
    
- **308**: Permanent redirect인데, **요청 메소드도 유지**돼.
    

---

### Next.js의 `redirect()` 함수는 그래서

- 임시 리다이렉트 → `307` 사용 (POST → POST 그대로)
    
- 영구 리다이렉트 → `308` 사용 (POST → POST 그대로)
    

**기본값이 307**인 이유는, **요청 방식(method)을 보존**하는 게 안전하기 때문이야.

---

### 결론

> **Next.js의 `redirect()`는 요청을 안전하게 유지(POST는 POST, GET은 GET)하도록 기본적으로 307/308을 사용한다.**  
> 302/301은 요청 메소드가 바뀔 수 있어서 쓰지 않는다.

## authInterrupts
- 실험단계
- 사용하려면 next.config.js에서 `authInterrupts` 옵션 활성화
- The `authInterrupts` configuration 옵션을 사용하면 [`forbidden`](https://nextjs.org/docs/app/api-reference/functions/forbidden) , [`unauthorized`](https://nextjs.org/docs/app/api-reference/functions/unauthorized) API를 사용할 수 있습니다. 
- [`forbidden.js`](https://nextjs.org/docs/app/api-reference/file-conventions/forbidden), [`unauthorized.js`](https://nextjs.org/docs/app/api-reference/file-conventions/unauthorized) 파일 컨벤션으로 UI 커스텀 가능
```ts
import type { NextConfig } from 'next'
 
const nextConfig: NextConfig = {
  experimental: {
    authInterrupts: true,
  },
}
 
export default nextConfig
```

### forbidden
- https://github.com/vercel/next.js/blob/e1b23eb16af257cce80ab7c6893617ff34b65bd9/packages/next/src/client/components/forbidden.ts
```ts
import {
  HTTP_ERROR_FALLBACK_ERROR_CODE,
  type HTTPAccessFallbackError,
} from './http-access-fallback/http-access-fallback'

// TODO: Add `forbidden` docs
/**
 * @experimental
 * This function allows you to render the [forbidden.js file](https://nextjs.org/docs/app/api-reference/file-conventions/forbidden)
 * within a route segment as well as inject a tag.
 *
 * `forbidden()` can be used in
 * [Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components),
 * [Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers), and
 * [Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations).
 *
 * Read more: [Next.js Docs: `forbidden`](https://nextjs.org/docs/app/api-reference/functions/forbidden)
 */

const DIGEST = `${HTTP_ERROR_FALLBACK_ERROR_CODE};403`

export function forbidden(): never {
// next.config에 authInterrupts이 활성화되어 있지 않으면 에러 throw
  if (!process.env.__NEXT_EXPERIMENTAL_AUTH_INTERRUPTS) {
    throw new Error(
      `\`forbidden()\` is experimental and only allowed to be enabled when \`experimental.authInterrupts\` is enabled.`
    )
  }

  // eslint-disable-next-line no-throw-literal
  const error = new Error(DIGEST) as HTTPAccessFallbackError
  ;(error as HTTPAccessFallbackError).digest = DIGEST
  throw error
  // Error를 Throw하는데 
}
```
### unauthorized
- https://github.com/vercel/next.js/blob/e1b23eb16af257cce80ab7c6893617ff34b65bd9/packages/next/src/client/components/unauthorized.ts
```ts
import {
  HTTP_ERROR_FALLBACK_ERROR_CODE,
  type HTTPAccessFallbackError,
} from './http-access-fallback/http-access-fallback'

// TODO: Add `unauthorized` docs
/**
 * @experimental
 * This function allows you to render the [unauthorized.js file](https://nextjs.org/docs/app/api-reference/file-conventions/unauthorized)
 * within a route segment as well as inject a tag.
 *
 * `unauthorized()` can be used in
 * [Server Components](https://nextjs.org/docs/app/building-your-application/rendering/server-components),
 * [Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers), and
 * [Server Actions](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations).
 *
 *
 * Read more: [Next.js Docs: `unauthorized`](https://nextjs.org/docs/app/api-reference/functions/unauthorized)
 */

const DIGEST = `${HTTP_ERROR_FALLBACK_ERROR_CODE};401`

export function unauthorized(): never {
  if (!process.env.__NEXT_EXPERIMENTAL_AUTH_INTERRUPTS) {
    throw new Error(
      `\`unauthorized()\` is experimental and only allowed to be used when \`experimental.authInterrupts\` is enabled.`
    )
  }

  // eslint-disable-next-line no-throw-literal
  const error = new Error(DIGEST) as HTTPAccessFallbackError
  ;(error as HTTPAccessFallbackError).digest = DIGEST
  throw error
}
```


이 에러바운더리에 의해
https://github.com/vercel/next.js/blob/0b136b273e6e7f8727d591209111b1d949e62945/packages/next/src/client/components/http-access-fallback/error-boundary.tsx

Forbidden컴포넌트가 렌더링
https://github.com/vercel/next.js/blob/37f26ac2892d480c4e2570d6bc6d1f3c0bf8dc81/packages/next/src/client/components/forbidden-error.tsx
```tsx
import { HTTPAccessErrorFallback } from './http-access-fallback/error-fallback'

export default function Forbidden() {
  return (
    <HTTPAccessErrorFallback
      status={403}
      message="This page could not be accessed."
    />
  )
}
```

Unauthorized 컴포넌트가 렌더링
https://github.com/vercel/next.js/blob/37f26ac2892d480c4e2570d6bc6d1f3c0bf8dc81/packages/next/src/client/components/unauthorized-error.tsx
``
```tsx
import { HTTPAccessErrorFallback } from './http-access-fallback/error-fallback'

export default function Unauthorized() {
  return (
    <HTTPAccessErrorFallback
      status={401}
      message="You're not authorized to access this page."
    />
  )
}
```

Next.js의 에러 바운더리는 앱의 최상위 레벨에서 자동으로 설정
구체적으로는 [`packages/next/src/client/components/app-router.tsx`](https://github.com/vercel/next.js/blob/527fe7a0565e4896b7baa7365dbfb5a08f7fd730/packages/next/src/client/components/app-router.tsx)에서 설정됨
