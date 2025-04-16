# w03. <link>

소개할 순서는 아래와 같음

1. Link이동 시 어떻게 상태를 보존하는가
2. prefetch

## Link이동 시 어떻게 상태를 보존하는가

해당 개념을 설명하기 전 우선 `하드 네비게이션 & 소프트 네비게이션`을 훑어봄

### 하드 네비게이션

흔히 하드네비게이션이라 부르는것은 아래와 같음

```typescript
// 1. 일반 <a> 태그
<a href='/about'>About</a>;

// 2. window.location
window.location.href = '/about';
```

즉 전체 페이지가 새로고침 되며, 모든 상태가 초기화됨.

### 소프트 네비게이션

```typescript
<Link href='/about'>About</Link>;
// - 클라이언트 사이드 라우팅
// - 상태 유지
// - 부분 업데이트

// 2. useRouter
const router = useRouter();
router.push('/about');
```

파셜 렌더링으로 페이지 이동

route segments단위

소프트 네비게이션인 `<Link>` 사용 시
Link 컴포넌트는 기본적으로 viewport에 보이는 링크들을 자동으로 prefetch합니다.

네비 -> 유저 클라
라우팅 -> 클라 -> 서버

1. code splitting
   서버에서 route segments 단위로 번들을 쪼개서 응답해줌
2. prefetching
   rsc payload를 가져옴
3. caching
   요청 최적화, layout등 캐시함
4. 파셜 렌더링 진행

usage

```tsx
import Link from 'next/link';

export default function Page() {
  return <Link href='/dashboard'>Dashboard</Link>;
}
```

여기서 이제 링크 컴포넌트가 시작됩니다.

```typescript
export default function LinkComponent(
  props: LinkProps & {
    children: React.ReactNode
    ref: React.Ref<HTMLAnchorElement>
  }
) {
  ...
}
```

> `<Link>`는 HTML `<a>` 요소를 확장하여 prefetching 및 클라이언트 사이드 라우팅을 제공하는 Client Component입니다.

이후

```typescript
const router =  React.useContext(AppRouterContext)
로 라우터 객체를 가져옵니다.
```

그리구 뷰포트에 감지되는 link만 프리패치하니깐 감지 코드가 필요한데 아래와 같습니다.

```typescript
const observeLinkVisibilityOnMount = React.useCallback(
  (element: HTMLAnchorElement | SVGAElement) => {
    if (router !== null) {
      linkInstanceRef.current = mountLinkInstance(element, href, router, appPrefetchKind, prefetchEnabled, setOptimisticLinkStatus);
    }
    return () => {
      // cleanup 함수
      if (linkInstanceRef.current) {
        unmountLinkForCurrentNavigation(linkInstanceRef.current);
        linkInstanceRef.current = null;
      }
      unmountPrefetchableInstance(element);
    };
  },
  [prefetchEnabled, href, router, appPrefetchKind, setOptimisticLinkStatus]
);
const mergedRef = useMergedRef(observeLinkVisibilityOnMount, childRef)
...
const childProps: {
  onTouchStart?: React.TouchEventHandler<HTMLAnchorElement>
  onMouseEnter: React.MouseEventHandler<HTMLAnchorElement>
  onClick: React.MouseEventHandler<HTMLAnchorElement>
  href?: string
  ref?: any
} = {
  ref: mergedRef,
}
...
link = (
  <a {...restProps} {...childProps}>
    {children}
  </a>
)

return (
<LinkStatusContext.Provider value={linkStatus}>
  {link}
</LinkStatusContext.Provider>
)
```

```typescript
React.useEffect(() => {
  // in dev, we only prefetch on hover to avoid wasting resources as the prefetch will trigger compiling the page.
  if (process.env.NODE_ENV !== 'production') {
    return;
  }

  if (!router) {
    return;
  }

  // If we don't need to prefetch the URL, don't do prefetch.
  if (!isVisible || !prefetchEnabled) {
    return;
  }

  // Prefetch the URL.
  prefetch(router, href, as, { locale });
}, [as, href, isVisible, locale, prefetchEnabled, router?.locale, router]);
```

해당 링크를 클릭하는 액션은 아래와 같습니다.

```typescript
//link.tsx
onClick(e) {
...
  linkClicked(
    e,
    router,
    href,
    as,
    replace,
    shallow,
    scroll,
    locale,
    onNavigate
  )
},

function linkClicked(...){
  ...

const navigate = () => {
    if (onNavigate) {
      let isDefaultPrevented = false

      onNavigate({
        preventDefault: () => {
          isDefaultPrevented = true
        },
      })

      if (isDefaultPrevented) {
        return
      }
    }

    dispatchNavigateAction(
      as || href,
      replace ? 'replace' : 'push',
      scroll ?? true,
      linkInstanceRef.current
    )
  }

  e.preventDefault();
  React.startTransition(navigate)
}

//app-router-instance.ts
export function dispatchNavigateAction(...){
  ...
    dispatchAppRouterAction({
      type: ACTION_NAVIGATE,
      url,
      isExternalUrl: isExternalURL(url),
      locationSearch: location.search,
      shouldScroll,
      navigateType,
      allowAliasing: true,
    })
}

//ppr-navigations.ts
const task = startPPRNavigation(
    navigatedAt,
    currentCache,
    currentTree,
    treePatch,
    seedData,
    head,
    isHeadPartial,
    false,
    scrollableSegments
)
```

결국 프로덕션환경에서 startPPRNavigation에 도달하여, 파셜렌더링이 진행되며 이 메서드 안에서는 캐시된 노드등을 불러오며 소프트네비게이션을 진행합니다.

결론
기존 a태그 동작을 막고, 파셜 네비게이션을 통해 소프트 네비게이션 실행

최종적으로 렌더링하는 HTML를 반환합니다.

```typescript
return (
  <a {...restProps} {...childProps}>
    {children}
  </a>
);
```

이제 해당 link가 보이면 아래 코드에 등록된 함수가 실행됩니다.

```typescript
linkInstanceRef.current = mountLinkInstance(
...
  )

// netx/src/client/components/links.ts
mountLinkInstance() {
  if (prefetchEnabled) {
    const prefetchURL = coercePrefetchableUrl(href)
    if (prefetchURL !== null) {
      const instance: PrefetchableLinkInstance = {
        router,
        kind,
        isVisible: false,
        wasHoveredOrTouched: false,
        prefetchTask: null,
        cacheVersion: -1,
        prefetchHref: prefetchURL.href,
        setOptimisticLinkStatus,
      }
      // We only observe the link's visibility if it's prefetchable. For
      // example, this excludes links to external URLs.
      observeVisibility(element, instance)
      return instance
    }
  }
...
}

//가시성 걸리면 handleIntersect 실행
const observer: IntersectionObserver | null =
  typeof IntersectionObserver === 'function'
    ? new IntersectionObserver(handleIntersect, {

        rootMargin: '200px',
      })
    : null


function observeVisibility(element: Element, instance: PrefetchableInstance) {
  const existingInstance = prefetchable.get(element)
  if (existingInstance !== undefined) {
    // This shouldn't happen because each <Link> component should have its own
    // anchor tag instance, but it's defensive coding to avoid a memory leak in
    // case there's a logical error somewhere else.
    unmountPrefetchableInstance(element)
  }
  // Only track prefetchable links that have a valid prefetch URL
  prefetchable.set(element, instance)
  if (observer !== null) {
    observer.observe(element)
  }
}

//handleIntersect -> onLinkVisibilityChanged -> rescheduleLinkPrefetch -> cancelPrefetchTask -> ..
```

결론

- 정적 라우트는 뷰포트 진입 시 전체 페이지(HTML, RSC, 데이터)를 프리패치합니다.
- 동적 라우트는 뷰포트 진입 시 RSC 페이로드만 프리패치하여 초기 로딩을 최적화합니다.
- 마우스 오버나 터치 이벤트 발생 시, 동적 라우트도 전체 페이지를 프리패치합니다.
- 프리패치 우선순위는 사용자 상호작용이 있는 경우 더 높게 설정됩니다.
- 프리패치된 데이터는 캐시에 저장되어 실제 네비게이션 시 재사용됩니다.
- 개발 환경에서는 프리패치가 비활성화되어 있습니다.
- 프리패치 종류는 PrefetchKind.AUTO(기본)와 PrefetchKind.FULL로 구분됩니다.
- 동적 라우트의 경우 loading.js가 있는 가장 가까운 부모 세그먼트까지만 부분 프리패치됩니다.
- 프리패치된 데이터는 캐시 버전으로 관리되어 오래된 데이터 사용을 방지합니다.
- 이 모든 메커니즘은 사용자 경험을 최적화하고 페이지 전환을 더 빠르게 만듭니다.
