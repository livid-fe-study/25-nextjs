# w09. cache

## Request Memoization

- react의 cache API를 이용

```ts
// patch-fetch.ts

// we patch fetch to collect cache information used for
// determining if a page is static or not
export function patchFetch(options: PatchableModule) {
  // If we've already patched fetch, we should not patch it again.
  if (isFetchPatched()) return

  // Grab the original fetch function. We'll attach this so we can use it in
  // the patched fetch function.
  const original = createDedupeFetch(globalThis.fetch)

  // Set the global fetch to the patched fetch.
  globalThis.fetch = createPatchedFetcher(original, options)
}
```

```ts
// dedupe-fetch.ts

export function createDedupeFetch(originalFetch: typeof fetch) {
    // url is cache key
    const getCacheEntries = React.cache(
      (url: string): CacheEntry[] => []
    )
    
    let cacheKey = generateCacheKey(request)
    let url = request.url
  
    // 캐시돼있는 경우
    // cacheEntries에 있는 캐시된 promise를 사용
    const cacheEntries = getCacheEntries(url)
    for (let i = 0, j = cacheEntries.length; i < j; i += 1) {
      const [key, promise] = cacheEntries[i]
      if (key === cacheKey) {
        return promise.then(() => {
          const response = cacheEntries[i][2]
          return response
        })
      }
    }
    
    // 캐시 없는 경우
    const promise = originalFetch(resource, options)
    const entry: CacheEntry = [cacheKey, promise, null]
    cacheEntries.push(entry)

    return promise.then((response) => {
      return response
    })
}
```

## Data Cache

- workStore의 incrementalCache에 요청 캐시를 저장

```ts
// patch-fetch.ts

export function createPatchedFetcher(
  originFetch: Fetcher,
  { workAsyncStorage, workUnitAsyncStorage }: PatchableModule
): PatchedFetcher {
    
    // ...

    // workStore에서 incrementalCache를 가져옴.
    let cacheKey: string | undefined
    const { incrementalCache } = workStore
    
    if (
      incrementalCache &&
      (isCacheableRevalidate ||
        useCacheOrRequestStore?.serverComponentsHmrCache)
    ) {
      try {
        // 캐시 키를 만듬
        cacheKey = await incrementalCache.generateCacheKey(
          fetchUrl,
          isRequestInput ? (input as RequestInit) : init
        )
      }
    }


    // ...

    // 실제 fetch 요청 처리하는 함수
    const doOriginalFetch = async (
      isStale?: boolean,
      cacheReasonOverride?: string
    ) => {
        // ...
        
        return originFetch(input, clonedInit)
            .then(async (res) => {
                // ...
            
                // 오리지널 fetch 요청이 성공하면
                if (
                    res.status === 200 &&
                    incrementalCache &&
                    cacheKey &&
                    (isCacheableRevalidate ||
                      useCacheOrRequestStore?.serverComponentsHmrCache)
                  ) {
                      // 캐시에 저장
                      await incrementalCache.set(
                        cacheKey,
                        {
                          kind: CachedRouteKind.FETCH,
                          data: fetchedData,
                          revalidate: normalizedRevalidate,
                        },
                        { fetchCache: true, fetchUrl, fetchIdx, tags }
                      )
                      await handleUnlock()
                }
            
            // ...
        })
    }
    
    
    // ...
    
    if (cacheKey && incrementalCache) {
        let cachedFetchData: CachedFetchData | undefined
        
        // ...
        
        // revaliddate가 0 이상인 경우
        if (isCacheableRevalidate && !cachedFetchData) {
            const entry = workStore.isOnDemandRevalidate
              ? null
              : await incrementalCache.get(cacheKey, {
                  kind: IncrementalCacheKind.FETCH,
                  revalidate: finalRevalidate,
                  fetchUrl,
                  fetchIdx,
                  tags,
                  softTags: implicitTags?.tags,
                })
            
            // 캐시가 stale 상태면 백그라운드에서 refetch한다.
            // cachedFetchData에 캐시된 데이터를 담는다.
            if (entry?.value && entry.value.kind === CachedRouteKind.FETCH) {
              // when stale and is revalidating we wait for fresh data
              // so the revalidated entry has the updated data
              if (workStore.isRevalidate && entry.isStale) {
                isForegroundRevalidate = true
              } else {
                if (entry.isStale) {
                  workStore.pendingRevalidates ??= {}
                  if (!workStore.pendingRevalidates[cacheKey]) {
                    const pendingRevalidate = doOriginalFetch(true)
                      .then(async (response) => ({
                        body: await response.arrayBuffer(),
                        headers: response.headers,
                        status: response.status,
                        statusText: response.statusText,
                      }))
                      .finally(() => {
                        workStore.pendingRevalidates ??= {}
                        delete workStore.pendingRevalidates[cacheKey || '']
                      })

                    // Attach the empty catch here so we don't get a "unhandled
                    // promise rejection" warning.
                    pendingRevalidate.catch(console.error)

                    workStore.pendingRevalidates[cacheKey] = pendingRevalidate
                  }
                }

                cachedFetchData = entry.value.data
              }
            }
            
            // 캐시된 응답 반환
            if (cachedFetchData) {
                const response = new Response(
                  Buffer.from(cachedFetchData.body, 'base64'),
                  {
                    headers: cachedFetchData.headers,
                    status: cachedFetchData.status,
                  }
                )

                Object.defineProperty(response, 'url', {
                  value: cachedFetchData.url,
                })

                return response
              }
        }
    }
        
    // SSG 일 때 처리
    if (workStore.isStaticGeneration && init && typeof init === 'object') {
        // ...
    }
    
    // foreground revalidate (on-demand revalidation?)
    if (cacheKey && isForegroundRevalidate) {
        // ...
    } else {
      return doOriginalFetch(false, cacheReasonOverride)
    }
    
```            

## Full Route Cache

- response-cache 활용
- base-servet.ts -> renderToResponseWithComponentsImpl -> doRender

```ts
// If this is a app router page, PPR is enabled, and PFPR is also
// enabled, then we should use the fallback renderer.
else if (
  isRoutePPREnabled &&
  isAppPageRouteModule(components.routeModule) &&
  !isRSCRequest
) {
  // We use the response cache here to handle the revalidation and
  // management of the fallback shell.
  fallbackResponse = await this.responseCache.get(
    isProduction ? pathname : null,
    // This is the response generator for the fallback shell.
    async () =>
      doRender({
        // We pass `undefined` as rendering a fallback isn't resumed
        // here.
        postponed: undefined,
        pagesFallback: undefined,
        fallbackRouteParams:
          // If we're in production of we're debugging the fallback
          // shell then we should postpone when dynamic params are
          // accessed.
          isProduction || isDebugFallbackShell
            ? getFallbackRouteParams(pathname)
            : null,
      }),
    {
      routeKind: RouteKind.APP_PAGE,
      incrementalCache,
      isRoutePPREnabled,
      isFallback: true,
    }
  )
}
```

## Client-side Router Cache

- 네비게이션 로직이 reducer 형태로 구현돼있다.
- client segment cache 기능이 실험 기능으로 구현돼 있는 듯.

```ts
  // navigate-reducer.ts

  if (process.env.__NEXT_CLIENT_SEGMENT_CACHE) {
    // (Very Early Experimental Feature) Segment Cache
    //
    // Bypass the normal prefetch cache and use the new per-segment cache
    // implementation instead. This is only supported if PPR is enabled, too.
    //
    // Temporary glue code between the router reducer and the new navigation
    // implementation. Eventually we'll rewrite the router reducer to a
    // state machine.
    const result = navigateUsingSegmentCache(
      url,
      state.cache,
      state.tree,
      state.nextUrl,
      shouldScroll
    )
    return handleNavigationResult(url, state, mutable, pendingPush, result)
  }
```

## BF Cache

- BFCache 옵션으로 활성화

```ts
// bfcache.ts

// 최근에 활성화된 N개의 트리를 관리하는 훅?
export function useRouterBFCache(
  activeTree: FlightRouterState,
  activeStateKey: string
): RouterBFCacheEntry {
    // ...
}
```

```tsx
// layout-router.tsx

// bfcacheEntry의 linked-list를 만들어서, 해당 리스트를 Activity로 렌더링한다.
// 현재 활성화된 트리만 visible로 설정
    if (process.env.__NEXT_ROUTER_BF_CACHE) {
      child = (
        <Activity
          key={stateKey}
          mode={stateKey === activeStateKey ? 'visible' : 'hidden'}
        >
          {child}
        </Activity>
      )
    }
```

