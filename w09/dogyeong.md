# w08. middleware

- next-middleware-loader
- middleware-plugin
- next-server.ts 파일에서 runMiddleware 함수

### nodejs 세팅

```ts
// next.config.js
const nextConfig = {
  experimental: {
    nodeMiddleware: true,
  },
};

module.exports = nextConfig;
```

```ts
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
 
export function middleware(request: NextRequest) {
  return NextResponse.next()
}

export const config = {
  matcher: ["/"],
  runtime: 'nodejs'
};
```

### 서버에서 미들웨어 실행

```ts
// next-server.ts
export default class NextNodeServer {
    protected async runMiddleware() {
        const middleware = await this.getMiddleware()
        // {
        //   match: [Function (anonymous)],
        //   page: '/',
        //   matchers: [
        //     {
        //       regexp: '^(?:\\/(_next\\/data\\/[^/]{1,}))?(?:\\/(\\/?index|\\/?index\\.json))?[\\/#\\?]?$',
        //       originalSource: '/'
        //     }
        //   ]
        // }
        
        const middlewareInfo = this.getEdgeFunctionInfo({
          page: middleware.page,
          middleware: true,
        })
        
        const method = /* ... */
        const requestData = { /* ... */ }
        
        // nodejs runtime일 때, middlewareInfo === null
        if (!middlewareInfo) {
            let middlewareModule
            
            // require('dist/server/middleware.js')
            middlewareModule = await this.loadNodeMiddleware()
            
            const adapterFn: typeof import('./web/adapter').adapter =
                middlewareModule.default || middlewareModule
            
            result = await adapterFn({
                handler: middlewareModule.middleware || middlewareModule,
                request: {
                    ...requestData,
                    body: hasRequestBody
                        ? requestData.body.cloneBodyStream()
                        : undefined,
                },
                page: 'middleware',
            })
            
            if (hasRequestBody) {
                requestData.body.finalize()
            }
        // edge runtime 일 때
        } else {
            // ...
        }
             
    }
}

```

- 미들웨어 테스트
  - https://nextjs.org/docs/app/building-your-application/routing/middleware#unit-testing-experimental
