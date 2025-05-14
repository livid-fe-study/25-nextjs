# w07. script, font

## script 

### onReady, onLoad

```ts
  const hasOnReadyEffectCalled = useRef(false)

  // 스크립트가 로드된 후, 다시 마운트될 때 onReady를 호출하기 위한 이펙트
  useEffect(() => {
    const cacheKey = id || src
    if (!hasOnReadyEffectCalled.current) {
      // Run onReady if script has loaded before but component is re-mounted
      if (onReady && cacheKey && LoadCache.has(cacheKey)) {
        onReady()
      }

      hasOnReadyEffectCalled.current = true
    }
  }, [onReady, id, src])

  const hasLoadScriptEffectCalled = useRef(false)

  // 마운트될 때 마다 loadScript를 호출하는 이펙트
  // loadScript가 여러 번 호출되면 첫 번째 호출만 동작한다. (두 번째부터 스킵)
  useEffect(() => {
    if (!hasLoadScriptEffectCalled.current) {
      if (strategy === 'afterInteractive') {
        loadScript(props)
      } else if (strategy === 'lazyOnload') {
        loadLazyScript(props)
      }

      hasLoadScriptEffectCalled.current = true
    }
  }, [props, strategy])
```
### loadScript

- script 태그를 만들어서 load 이벤트 핸들러로 처리한다.

```ts
  const el = document.createElement('script')

  const loadPromise = new Promise<void>((resolve, reject) => {
    el.addEventListener('load', function (e) {
      resolve()
      if (onLoad) {
        onLoad.call(this, e)
      }
      afterLoad()
    })
    el.addEventListener('error', function (e) {
      reject(e)
    })
  }).catch(function (e) {
    if (onError) {
      onError(e)
    }
  })
```

### strategy: 'beforeInteractive'

- `<Script>` 컴포넌트 렌더링할 때 바로 `loadScript` 호출

### strategy: 'afterInteractive'

- `<Script>` 컴포넌트의 useEffect에서 `loadScript` 호출

### strategy: 'lazyOnLoad'

- `<Script>` 컴포넌트의 useEffect에서 `loadLazyScript` 호출
- document load 후, idle일 때 로딩한다.

```ts
function loadLazyScript(props: ScriptProps) {
  if (document.readyState === 'complete') {
    requestIdleCallback(() => loadScript(props))
  } else {
    window.addEventListener('load', () => {
      requestIdleCallback(() => loadScript(props))
    })
  }
}
```

### strategy: 'worker'

- `'beforeInteractive'` 처럼 바로 loadScript를 호출하지만, [`partytown`](https://partytown.qwik.dev/)을 써서 worker로 로드한다. 



## font

- 컴파일 과정에서 font 함수 호출을 처리하는듯?
- loader.ts 파일을 보면 웹팩 로더 설정이 있다.
- 구글 폰트 위주로..

### fetch font declarations

- 폰트 메타데이터는 `font-data.json`에 있음
- 메타데이터를 바탕으로 폰트 정보(url, fontFamily 등)를 얻을 수 있음

```ts!
    // Fetch CSS from Google Fonts or get it from the cache
    let fontFaceDeclarations = hasCachedCSS
      ? cssCache.get(url)
      : await fetchCSSFromGoogleFonts(url, fontFamily, isDev).catch((err) => {
          console.error(err)
          return null
        })
    if (!hasCachedCSS) {
      cssCache.set(url, fontFaceDeclarations ?? null)
    } else {
      cssCache.delete(url)
    }
```

### fetch font files

- 폰트 파일(.woff, .eot, .ttf 등)을 받아서 로컬에 저장한다.
- `.next/static/media` 에 저장됨

```ts
    // Download the font files extracted from the CSS
    const downloadedFiles = await Promise.all(
      fontFiles.map(async ({ googleFontFileUrl, preloadFontFile }) => {
        const hasCachedFont = fontCache.has(googleFontFileUrl)
        // Download the font file or get it from cache
        const fontFileBuffer = hasCachedFont
          ? fontCache.get(googleFontFileUrl)
          : await fetchFontFile(googleFontFileUrl, isDev).catch((err) => {
              console.error(err)
              return null
            })
        if (!hasCachedFont) {
          fontCache.set(googleFontFileUrl, fontFileBuffer ?? null)
        } else {
          fontCache.delete(googleFontFileUrl)
        }
        if (fontFileBuffer == null) {
          nextFontError(`Failed to fetch \`${fontFamily}\` from Google Fonts.`)
        }

        const ext = /\.(woff|woff2|eot|ttf|otf)$/.exec(googleFontFileUrl)![1]
        // Emit font file to .next/static/media
        const selfHostedFileUrl = emitFontFile(
          fontFileBuffer,
          ext,
          preloadFontFile,
          !!adjustFontFallbackMetrics
        )

        return {
          googleFontFileUrl,
          selfHostedFileUrl,
        }
      })
    )
```

### replace source

- self hosting 하는 리소스 경로로 바꿔치기

```ts
    /**
     * Replace the @font-face sources with the self-hosted files we just downloaded to .next/static/media
     *
     * E.g.
     * @font-face {
     *   font-family: 'Inter';
     *   src: url(https://fonts.gstatic.com/...) -> url(/_next/static/media/_.woff2)
     * }
     */
    let updatedCssResponse = fontFaceDeclarations
    for (const { googleFontFileUrl, selfHostedFileUrl } of downloadedFiles) {
      updatedCssResponse = updatedCssResponse.replace(
        new RegExp(escapeStringRegexp(googleFontFileUrl), 'g'),
        selfHostedFileUrl
      )
    }
```

### adjust font fallback

- `adjustFontFallback` 옵션이 있음
- CLS를 줄이기 위해 자동으로 폴백 폰트를 사용할지 여부
- `next/font/local`에서는 안되고 `'Arial'`, `'Times New Roman'`, `false` 만 가능

```ts
  // Get precalculated fallback font metrics, used to generate the fallback font CSS
  const adjustFontFallbackMetrics: AdjustFontFallback | undefined =
    adjustFontFallback ? getFallbackFontOverrideMetrics(fontFamily) : undefined
```

```ts
/**
 * Get precalculated fallback font metrics for the Google Fonts family.
 *
 * TODO:
 * We might want to calculate these values with fontkit instead (like in next/font/local).
 * That way we don't have to update the precalculated values every time a new font is added to Google Fonts.
 */
export function getFallbackFontOverrideMetrics(fontFamily: string) {
  try {
    const { ascent, descent, lineGap, fallbackFont, sizeAdjust } =
      calculateSizeAdjustValues(fontFamily)
    return {
      fallbackFont,
      ascentOverride: `${ascent}%`,
      descentOverride: `${descent}%`,
      lineGapOverride: `${lineGap}%`,
      sizeAdjust: `${sizeAdjust}%`,
    }
  } catch {
    Log.error(`Failed to find font override values for font \`${fontFamily}\``)
  }
}
```

- 폰트 매트릭 정보를 제공하는 패키지가 있음
    - [@capsizecss/metrics](https://raw.githubusercontent.com/seek-oss/capsize/refs/heads/master/packages/metrics/src/entireMetricsCollection.json)
    - 요걸로 폰트 크기를 가져옴
    - ascent-override
    - descent-override
    - line-gap-override
    - size-adjust

### local font

- 사용한 폰트 중에서 fallback 생성에 사용할 폰트 선택
    - weight 400 에 가장 가까운 폰트 선택
    - 같은 weight인 경우, 이태릭보다 일반 스타일 선택
    - 같은 거리인 경우, 더 얇은 폰트 선택
- fallback metric 생성
    - 

```ts
  // Calculate the fallback font override values using the font file metadata
  let adjustFontFallbackMetrics: AdjustFontFallback | undefined
  if (adjustFontFallback !== false) {
    const fallbackFontFile = pickFontFileForFallbackGeneration(fontFiles)
    if (fallbackFontFile.fontMetadata) {
      adjustFontFallbackMetrics = getFallbackMetricsFromFontFile(
        fallbackFontFile.fontMetadata,
        adjustFontFallback === 'Times New Roman' ? 'serif' : 'sans-serif'
      )
    }
  }
```
