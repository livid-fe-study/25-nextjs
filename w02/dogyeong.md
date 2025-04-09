# w02. next/head, metadata

## next 개발환경 실행

```
git clone --depth=1 https://github.com/vercel/next.js.git

cd next.js

corepack enable pnpm

pnpm install

pnpm dev

pnpm next examples/hello-world

# pnpm next가 안돼서 vscode debugger 사용
# https://github.com/vercel/next.js/blob/canary/contributing/core/vscode-debugger.md
```

## next/head

- 컴포넌트 위치
    - `src/shared/lib/head.tsx`
- 컨텍스트
    - `AmpStateContext`
        - 구글 AMP?
    - `HeadManagerContext`
        - head 업데이트 관리
- 렌더링 전에 한 번 전처리를 거친다.
    - `src/shared/lib/head.tsx`
        - `unique()`
        - `reduceComponents()`
- head 렌더링
    - `src/server/render.tsx`
        - `renderToHTMLImpl` -> `renderDocument`
    - `src/client/index.tsx`
        - `hydrate` -> `wrapApp` -> `AppContainer`

## metadata
- `src/server/app-render/app-render.tsx`
- `src/lib/metadata/metadata.tsx`
    - `createMetadataComponents()` -> `getResolvedMetadata` -> `getResolvedMetadataImpl` -> `renderMetadata` ->`resolveMetadata` -> `resolveMetadataItems` -> `resolveMetadataItemsImpl` -> `collectMetadata` -> `resolveStaticMetadata`, `getDefinedMetadata`
    - 정적/동적 메타데이터 처리
    - 
        ```ts
        // src/lib/metadata/metadata.tsx 
        // collectMetadata

        // export metadata = {};
        const staticFilesMetadata = await resolveStaticMetadata(tree[2], props)
        
        // export generateMetadata = () => {};
        const metadataExport = mod ? getDefinedMetadata(mod, props, { route }) : null
        ```
- 메타데이터 스트리밍 여부를 결정하는 기준
    ```ts
    // src/server/base-server.ts

    serveStreamingMetadata: shouldServeStreamingMetadata(
      ua,
      this.nextConfig.htmlLimitedBots
    ),
    ```
    - `src/server/lib/streaming-metadata.ts`
    - static generation 할 때는 user-agent가 없기 때문에 streamingMetadata 아님. (= blockingMetadata)
    - dynamic rendering 할 때는 user-agent가 bot일 때만 blockingMetadata
    - https://nextjs.org/docs/app/api-reference/config/next-config-js/htmlLimitedBots
        ```ts
        export function shouldServeStreamingMetadata(
          userAgent: string,
          htmlLimitedBots: string | undefined
        ): boolean {
          const blockingMetadataUARegex = new RegExp(
            htmlLimitedBots || HTML_LIMITED_BOT_UA_RE_STRING,
            'i'
          )
          return (
            // When it's static generation, userAgents are not available - do not serve streaming metadata
            !!userAgent && !blockingMetadataUARegex.test(userAgent)
          )
        }
        ```
- 스트리밍 여부에 따라 promise를 await한다.
    ```tsx
    const promise = resolveFinalMetadata()
    if (serveStreamingMetadata) {
      return (
        <Suspense fallback={null}>
          <AsyncMetadata promise={promise} />
        </Suspense>
      )
    }
    const metadataState = await promise
    return metadataState.metadata
    ```
