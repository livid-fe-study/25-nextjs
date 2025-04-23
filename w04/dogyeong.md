# w04. Image

```ts
// get-img-props.ts

// static image의 src prop 예시
{
  src: '/_next/static/media/avatartion.aaac18e4.png',
  height: 588,
  width: 640,
  blurDataURL: '/_next/image?url=%2F_next%2Fstatic%2Fmedia%2Favatartion.aaac18e4.png&w=8&q=70',
  blurWidth: 8,
  blurHeight: 7
}

getImgProps -> generateImgAttrs
```

리액트와 브라우저 동작과 관련된 코멘트

```ts
// get-img-props.ts

function generateImgAttrs({ ... }) {
  if (unoptimized) {
    return { src, srcSet: undefined, sizes: undefined }
  }

  const { widths, kind } = getWidths(config, width, sizes)
  const last = widths.length - 1

  return {
    sizes: !sizes && kind === 'w' ? '100vw' : sizes,
    srcSet: widths
      .map(
        (w, i) =>
          `${loader({ config, src, quality, width: w })} ${
            kind === 'w' ? w : i + 1
          }${kind}`
      )
      .join(', '),

    // It's intended to keep `src` the last attribute because React updates
    // attributes in order. If we keep `src` the first one, Safari will
    // immediately start to fetch `src`, before `sizes` and `srcSet` are even
    // updated by React. That causes multiple unnecessary requests if `srcSet`
    // and `sizes` are defined.
    // This bug cannot be reproduced in Chrome or Firefox.
    src: loader({ config, src, quality, width: widths[last] }),
  }
}
```

imageOptimizer = 이미지 최적화

- `next-server.ts` -> `handleNextImageRequest` -> `imageOptimizer` -> `optimizeImage`

`.next/cache/images` 디렉토리에 이미지를 생성한다. 
그 아래 폴더 명이 cacheKey. 

```ts
// next-server.ts
const imageOptimizerCache = new ImageOptimizerCache({
    distDir: this.distDir,
    nextConfig: this.nextConfig,
})

// image-optimizer.ts
export class ImageOptimizerCache {
    constructor() {
        // ...
        this.cacheDir = join(distDir, 'cache', 'images')
        // ...
    }
}
```

![image](https://hackmd.io/_uploads/SJGvyBMJle.png)

파일명은 여러 정보를 조합해서 만든다.

```ts
const [maxAgeSt, expireAtSt, etag, upstreamEtag, extension] = file.split('.', 5)

// maxAge: 60
// expireAt: 1745141399940
// etag: SHhXp_Gh1cslQaVRB4nyMxJM8Ju1HESSXRPTMX2X2No
// upstreamEtag: Vy8iNGIyNi0xOTY1MjdmYzg1MiI
// extension: webp
// 
// 60.1745141399940.SHhXp_Gh1cslQaVRB4nyMxJM8Ju1HESSXRPTMX2X2No.Vy8iNGIyNi0xOTY1MjdmYzg1MiI.webp
```

sharp 사용하는 부분

```ts
// image-optimizer.ts

export async function optimizeImage({ ... }): Promise<Buffer> {
  const sharp = getSharp(concurrency)
  const transformer = sharp(buffer, {
    limitInputPixels,
    sequentialRead: sequentialRead ?? undefined,
  })
    .timeout({
      seconds: timeoutInSeconds ?? 7,
    })
    .rotate()

  if (height) {
    transformer.resize(width, height)
  } else {
    transformer.resize(width, undefined, {
      withoutEnlargement: true,
    })
  }

  if (contentType === AVIF) {
    transformer.avif({
      quality: Math.max(quality - 20, 1),
      effort: 3,
    })
  } else if (contentType === WEBP) {
    transformer.webp({ quality })
  } else if (contentType === PNG) {
    transformer.png({ quality })
  } else if (contentType === JPEG) {
    transformer.jpeg({ quality, mozjpeg: true })
  }

  const optimizedBuffer = await transformer.toBuffer()

  return optimizedBuffer
}
```

```ts
// 배치 작업을 위함. (여러 요청을 하나의 promise로 처리, 중복 처리 제거)
// promise job들이 전부 실행된 직후에 콜백을 수행한다.
const scheduleOnNextTick = (cb) => {
  Promise.resolve().then(() => {
    if (process.env.NEXT_RUNTIME === 'edge') {
      setTimeout(cb, 0)
    } else {
      process.nextTick(cb)
    }
  })
}
```
