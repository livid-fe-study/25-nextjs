# w03

src/clinet/app-dir/link.tsx


useSearchParams, usePathname 훅은 단순히 현재 window.location.href 값을 반환한다.

```tsx
// app-router.tsx

const { searchParams, pathname } = useMemo(() => {
    const url = new URL(
      canonicalUrl,
      typeof window === 'undefined' ? 'http://n' : window.location.href
    )

    return {
      // This is turned into a readonly class in `useSearchParams`
      searchParams: url.searchParams,
      pathname: hasBasePath(url.pathname)
        ? removeBasePath(url.pathname)
        : url.pathname,
    }
}, [canonicalUrl])
```

```ts
// navigation.ts

export function useSearchParams(): ReadonlyURLSearchParams {
    const searchParams = useContext(SearchParamsContext)

    const readonlySearchParams = useMemo(() => {
    return new ReadonlyURLSearchParams(searchParams)
    }, [searchParams]) as ReadonlyURLSearchParams

    return readonlySearchParams
}

export function usePathname(): string {
    useDynamicRouteParams?.('usePathname()')

    return useContext(PathnameContext) as string
}
```


pushState, replaceState로 url이 변경되면 이것을 nextjs에도 반영하기 위해 원본 메서드를 patch한다.

```ts
// app-router.tsx

window.history.pushState = function pushState(
  data: any,
  _unused: string,
  url?: string | URL | null
): void {
  data = copyNextJsInternalHistoryState(data)

  if (url) {
    applyUrlFromHistoryPushReplace(url)
  }

  return originalPushState(data, _unused, url)
}
```

restore 액션을 dispatch한다.
restore 액션은 동기적으로 실행되며, pending 상태의 액션을 취소하고 바로 시작하도록 우선순위를 높인다.
결국 `applyUrlFromHistoryPushReplace`는 reducer로 들어가서 새로운 상태를 만들고, `setState(newState)` 를 호출해서 리렌더링한다. 

```tsx
// app-router-instance.ts

const actionQueue: AppRouterActionQueue = {
    state: initialState,
    dispatch: (payload: ReducerActions, setState: DispatchStatePromise) =>
      dispatchAction(actionQueue, payload, setState),
    action: async (state: AppRouterState, action: ReducerActions) => {
      const result = reducer(state, action)
      return result
    },
    pending: null,
    last: null,
    onRouterTransitionStart:
      instrumentationHooks !== null &&
      typeof instrumentationHooks.onRouterTransitionStart === 'function'
        ? instrumentationHooks.onRouterTransitionStart
        : null,
}

function dispatchAction(
  actionQueue: AppRouterActionQueue,
  payload: ReducerActions,
  setState: DispatchStatePromise
) {
  let resolvers: {
    resolve: (value: ReducerState) => void
    reject: (reason: any) => void
  } = { resolve: setState, reject: () => {} }

  // RESTORE 액션은 네비게이션이기 때문에 빨리 실행되어야 해서 동기적으로 수행하고, 
  // 나머지 액션은 비동기적으로 수행한다.
  if (payload.type !== ACTION_RESTORE) {
    const deferredPromise = new Promise<AppRouterState>((resolve, reject) => {
      resolvers = { resolve, reject }
    })

    startTransition(() => {
      setState(deferredPromise)
    })
  }

  const newAction: ActionQueueNode = {
    payload,
    next: null,
    resolve: resolvers.resolve,
    reject: resolvers.reject,
  }

  // 큐가 비어있으면
  if (actionQueue.pending === null) {
    actionQueue.last = newAction

    runAction({
      actionQueue,
      action: newAction,
      setState,
    })
  } else if (
    payload.type === ACTION_NAVIGATE ||
    payload.type === ACTION_RESTORE
  ) {
    // RESTORE, NAVIGATE 액션일 때, 큐에 액션이 있으면
    
    // 기존 pending 상태의 액션 취소
    actionQueue.pending.discarded = true

    // 그 뒤의 액션들은 이어서 수행
    newAction.next = actionQueue.pending.next

    if (actionQueue.pending.payload.type === ACTION_SERVER_ACTION) {
      actionQueue.needsRefresh = true
    }

    runAction({
      actionQueue,
      action: newAction,
      setState,
    })
  } else {
    if (actionQueue.last !== null) {
      actionQueue.last.next = newAction
    }
    actionQueue.last = newAction
  }
}
```
