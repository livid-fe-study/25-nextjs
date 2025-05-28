# next cache

## Next에서의 캐싱방식

1. Data
   - Request Memoization(Server)
   - Data Cache(Server)
2. Routing
   - Full Route Cache(Server)
   - Router Cache(Client)

### [Router Cache](https://nextjs.org/docs/app/deep-dive/caching#client-side-router-cache)(Client 측)

- RSC Payload에 대해서 세션 또는 정해진 시간동안 네비게이션 시 서버 요청 감소

### app-router.ts

```tsx
export type CacheNode = {
  lazyData: ReturnType<typeof createRecordFromThenable> | null;
  rsc: React.ReactNode | null;
  prefetchRsc: React.ReactNode | null;
  head: React.ReactNode | null;
  prefetchHead: React.ReactNode | null;
  parallelRoutes: Map<string, Map<string, CacheNode>>;
  loading: React.ReactNode | null;
  navigatedAt: number;
};

export function createEmptyCacheNode(): CacheNode {
  return {
    lazyData: null,
    rsc: null,
    prefetchRsc: null,
    head: null,
    prefetchHead: null,
    parallelRoutes: new Map(),
    loading: null,
    navigatedAt: -1,
  };
}
```

- **lazyData**: 서버 컴포넌트의 데이터를 비동기적으로 로드할 때 사용하는 thenable
- **rsc / prefetchRsc**: React 서버 컴포넌트 트리 / 미리 로드된 서버 컴포넌트
- **head / prefetchHead**: `<head>`에 넣을 요소들 (prefetchHead는 사전 로딩된 버전)
- **parallelRoutes**: 병렬 경로를 위한 중첩 라우팅 트리
- **loading**: 로딩 UI
- **navigatedAt**: 마지막으로 업데이트한 내비게이션의 타임스탬프

```tsx
const historyState = {
  __NA: true,
  __PRIVATE_NEXTJS_INTERNALS_TREE: tree,
};
window.history.pushState(historyState, "", canonicalUrl);
```

- `__PRIVATE_NEXTJS_INTERNALS_TREE`: 현재 라우팅 트리 (서버 컴포넌트 렌더링 정보 포함)
- `__NA`: App Router 사용 여부 플래그

> 이 값을 통해 [BFCache](https://kghworks.tistory.com/117) 복원 시에도 라우터 상태를 정확히 복구할 수 있습니다.

```tsx
function handlePageShow(event: PageTransitionEvent) {
  if (
    event.persisted &&
    window.history.state?.__PRIVATE_NEXTJS_INTERNALS_TREE
  ) {
    dispatchAppRouterAction({
      type: ACTION_RESTORE,
      url: new URL(window.location.href),
      tree: window.history.state.__PRIVATE_NEXTJS_INTERNALS_TREE,
    });
  }
}
```

- BFCache에서 복원될 때 `__PRIVATE_NEXTJS_INTERNALS_TREE`를 이용해 상태 복원

### Request Memoization, Data Cache

- 동일한 요청에 대한 중복 fetch를 방지

| 기능                           | 설명                               |
| ------------------------------ | ---------------------------------- |
| `createDedupeFetch`            | 중복 요청을 막는 fetch 생성기      |
| `generateCacheKey`             | 요청을 고유하게 식별하는 키 생성   |
| `React.cache`                  | 서버 컴포넌트에서 요청별 캐시 저장 |
| `cloneResponse`                | 버그 대응 및 캐시용 Response 복제  |
| `AbortSignal` 또는 `keepalive` | 캐시 제외 조건                     |
| `GET`, `HEAD` 요청만 캐싱      | 안전한 HTTP 메서드만 대상          |

### [dedupe-fetch.ts](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/lib/dedupe-fetch.ts)

```tsx
type CacheEntry = [
  key: string,
  promise: Promise<Response>,
  response: Response | null
];
```

- 저장되는 값들의 타입

### `generateCacheKey`

- 요청을 고유하게 식별하는 키 생성

```tsx
function generateCacheKey(request: Request): string {
  return JSON.stringify([
    request.method,
    Array.from(request.headers.entries()),
    request.mode,
    request.redirect,
    request.credentials,
    request.referrer,
    request.referrerPolicy,
    request.integrity,
  ]);
}
```

- `Request` 객체의 주요 속성을 문자열로 직렬화하여 **캐시 키를 생성**합니다.
- `body`는 포함되지 않으며, `GET`/`HEAD` 요청만 dedupe 대상입니다.
- 변경 가능성이 낮은 필드만 포함하여 **안정적인 키 생성**을 보장합니다.

### `createDedupeFetch`

- 기본 `fetch`를 감싸서 dedupe 기능이 적용된 새로운 `fetch` 함수 생성.

```tsx
const getCacheEntries = React.cache((url: string): CacheEntry[] => []);
```

- [cache](https://ko.react.dev/reference/react/cache)를 통해 가져온 url 정보를 캐싱합니다.

```tsx
if (options && options.signal) {
  return originalFetch(resource, options);
}
```

- `AbortController`의 `signal`이 있으면 **캐시하지 않고 그냥 요청**을 보냅니다.

url, cacheKey 정규화과정

```tsx
// Normalize the Request
let url: string;
let cacheKey: string;
if (typeof resource === "string" && !options) {
  // Fast path.
  cacheKey = simpleCacheKey;
  url = resource;
}
```

- `fetch('/api/data')`처럼 단순 문자열만 넘어온 경우 **빠른 경로(Fast path)** 처리.
- 이럴 경우엔 `Request` 객체를 생성하지 않고, 단순히 `url`만으로 캐시

```tsx
const request =
  typeof resource === "string" || resource instanceof URL
    ? new Request(resource, options)
    : resource;
```

- _resource_: _URL_ | _RequestInfo(string | Request)_
  - URL과 string일 경우엔 resoure와 option으로 새로운 Request 객체를 만듭니다.

```tsx
const cacheEntries = getCacheEntries(url)
for (...) {
  if (key === cacheKey) {
    return promise.then(() => {
      if (!response) throw new InvariantError('No cached response')
      const [cloned1, cloned2] = cloneResponse(response)
      cacheEntries[i][2] = cloned2
      return cloned1
    })
  }
}
```

- 같은 `url`에 대해 이미 캐시된 `CacheEntry[]`를 조회.
- `cacheKey`가 동일한 요청이 있으면:
  - `promise.then(...)`을 통해 응답이 준비되길 기다림.
  - 응답 객체는 스트림을 포함하기 때문에 반드시 `cloneResponse`로 **복제**해서 사용 (한 번만 읽을 수 있기 때문).
  - 복제된 응답 두 개 중 하나는 반환하고, 하나는 캐시에 다시 저장.
  찾는 키가 없다면(새로운 요청):
  ```tsx
  const promise = originalFetch(resource, options);
  const entry: CacheEntry = [cacheKey, promise, null];
  cacheEntries.push(entry);
  ```
  - 원래 fetch 수행.
  - `promise`와 함께 캐시에 저장.
  - 응답이 오면 `cloneResponse`를 통해 복제해서 하나는 캐시에 저장하고, 하나는 호출자에게 반환.

```tsx
const [cloned1, cloned2] = cloneResponse(response);
entry[2] = cloned2;
return cloned1;
```

`Response`는 한 번만 `.body`를 읽을 수 있으므로 **두 개로 복제**하여

- 하나는 사용자에게 전달
- 하나는 다음 사용자에게 재사용
