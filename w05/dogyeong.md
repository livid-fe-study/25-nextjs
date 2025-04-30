# w05. redirect, notFound, authInterrupts

## redirect

- redirect() 함수는 에러를 던진다.
- 에러를 던지기 때문에 try/catch 문 안에서 쓰면 에러를 다시 던져야 한다.

```ts
export function redirect(url, type) {
  type ??= actionAsyncStorage?.getStore()?.isAction
    ? RedirectType.push
    : RedirectType.replace

  throw getRedirectError(url, type, RedirectStatusCode.TemporaryRedirect)
}

export function getRedirectError(url, type, statusCode) {
  const error = new Error(REDIRECT_ERROR_CODE)
  error.digest = `${REDIRECT_ERROR_CODE};${type};${url};${statusCode};`
  return error
}
```

- redirect() 는 상태코드 307, permanentRedirect()는 상태코드 308으로 응답

- 서버 컴포넌트에서 동적으로 리디렉션 되는경우, 메타 태그를 추가해준다. 이 때의 상태코드는 200이 된다.

```ts
// make-get-server-inserted-html.tsx

const redirectUrl = addPathPrefix(
  getURLFromRedirectError(error),
  basePath
)
const statusCode = getRedirectStatusCodeFromError(error)
const isPermanent =
  statusCode === RedirectStatusCode.PermanentRedirect ? true : false
if (redirectUrl) {
  errorMetaTags.push(
    <meta
      id="__next-page-redirect"
      httpEquiv="refresh"
      content={`${isPermanent ? 0 : 1};url=${redirectUrl}`}
      key={error.digest}
    />
  )
}
```

- 클라이언트 측에서는 (아마도) 에러 바운더리로 잡아서 처리
- 아래 코드에서 startTransition을 쓴 것에 주목. 동기적으로 에러 처리하려면 트랜지션을 써야 한다.

```ts
function HandleRedirect({ redirect, reset, redirectType }) {
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

## notFound

- notFound() 함수는 에러를 던진다.

```ts
const DIGEST = `${HTTP_ERROR_FALLBACK_ERROR_CODE};404`

export function notFound(): never {
  const error = new Error(DIGEST) as HTTPAccessFallbackError
  ;(error as HTTPAccessFallbackError).digest = DIGEST

  throw error
}
```

- 에러를 잡으면 `<meta name="robots" content="noindex" />` 메타 태그를 추가한다.
- notFound, forbidden, unauthorized 에러를 체크할 때는 isHTTPAccessFallbackError를 사용한다.

```ts
if (isHTTPAccessFallbackError(error)) {
    errorMetaTags.push(
        <meta name="robots" content="noindex" key={error.digest} />,
        process.env.NODE_ENV === 'development' 
            ? <meta name="next-error" content="not-found" key="next-error" />
            : null
    )
}
```

## forbidden

- forbidden 함수도 에러를 던진다.

```ts
const DIGEST = `${HTTP_ERROR_FALLBACK_ERROR_CODE};403`

export function forbidden(): never {
  const error = new Error(DIGEST) as HTTPAccessFallbackError
  ;(error as HTTPAccessFallbackError).digest = DIGEST
  throw error
}
```

## unauthorized

- unauthorized 함수도 에러를 던진다.

```ts
const DIGEST = `${HTTP_ERROR_FALLBACK_ERROR_CODE};401`

export function unauthorized(): never {
  const error = new Error(DIGEST) as HTTPAccessFallbackError
  ;(error as HTTPAccessFallbackError).digest = DIGEST
  throw error
}
```

## 에러 핸들링

- 위 에러들을 잡는 에러 바운더리가 있다.
- HTTPAccessFallbackBoundary, RedirectBoundary

```ts
// layout-router.tsx

let child = (
  <TemplateContext.Provider
    key={stateKey}
    value={
      <ScrollAndFocusHandler segmentPath={segmentPath}>
        <ErrorBoundaryComponent
          errorComponent={error}
          errorStyles={errorStyles}
          errorScripts={errorScripts}
        >
          <LoadingBoundary loading={loadingModuleData}>
            <HTTPAccessFallbackBoundary
              notFound={notFound}
              forbidden={forbidden}
              unauthorized={unauthorized}
            >
              <RedirectBoundary>
                <InnerLayoutRouter
                  url={url}
                  tree={tree}
                  cacheNode={cacheNode}
                  segmentPath={segmentPath}
                />
              </RedirectBoundary>
            </HTTPAccessFallbackBoundary>
          </LoadingBoundary>
        </ErrorBoundaryComponent>
      </ScrollAndFocusHandler>
    }
  >
    {templateStyles}
    {templateScripts}
    {template}
  </TemplateContext.Provider>
)
```

- redirect는 router를 이용하고, 나머지는 지정된 컴포넌트를 렌더링한다.

```ts
render() {
    const { notFound, forbidden, unauthorized, children } = this.props
    const { triggeredStatus } = this.state
    const errorComponents = {
      [HTTPAccessErrorStatus.NOT_FOUND]: notFound,
      [HTTPAccessErrorStatus.FORBIDDEN]: forbidden,
      [HTTPAccessErrorStatus.UNAUTHORIZED]: unauthorized,
    }

    if (triggeredStatus) {
      const isNotFound =
        triggeredStatus === HTTPAccessErrorStatus.NOT_FOUND && notFound
      const isForbidden =
        triggeredStatus === HTTPAccessErrorStatus.FORBIDDEN && forbidden
      const isUnauthorized =
        triggeredStatus === HTTPAccessErrorStatus.UNAUTHORIZED && unauthorized

      // If there's no matched boundary in this layer, keep throwing the error by rendering the children
      if (!(isNotFound || isForbidden || isUnauthorized)) {
        return children
      }

      return (
        <>
          <meta name="robots" content="noindex" />
          {process.env.NODE_ENV === 'development' && (
            <meta
              name="boundary-next-error"
              content={getAccessFallbackErrorTypeByStatus(triggeredStatus)}
            />
          )}
          {errorComponents[triggeredStatus]}
        </>
      )
    }

    return children
}
```
