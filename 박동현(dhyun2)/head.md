# nextjs 톺아보기

## 서론

안녕하세요! 이번에 nextjs 톺아보기 스터디에 들어가게 되었는데요! 이번 주차 파트인 head를 정의부터 시작해서 react에서의 처리법, next의 처리법을 비교하며 nextjs의 head 동작원리에 다가가보겠습니다!

## 1\. 왜 는 중요한가?

는 와 다르게 컨텐츠로는 표현되지 않습니다.([What is the HTML head?](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Structuring_content/Webpage_metadata#what_is_the_html_head)) 대신에 페이지의 metadata를 정의하는 공간입니다.

### 1.1 head가 중요한 이유

대표적으로 아래와 같습니다

- SEO(검색 엔진 최적화): 검색 엔진이 HTML을 읽고 인덱싱할 때 참고하는 메타 정보  
   검색 엔진들은 title, description, robots, canonical 등을 중요하게 반응합니다.

  ```html
  <!-- 문서 제목 -->
  <title>프론트엔드 개발자를 위한 Head 완전정복</title>

  <!-- 페이지 설명 -->
  <meta name="description" content="Next.js와 HTML의 head 태그를 심층 분석하고 SEO 최적화 방법을 소개합니다." />

  <!-- 검색 엔진에 노출 허용 -->
  <meta name="robots" content="index, follow" />

  <!-- 작성자 정보 -->
  <meta name="author" content="박동현" />

  <!-- 사이트 정규 주소 (중복 방지) -->
  <link rel="canonical" href="https://yourdomain.com/head-guide" />
  ```

- OG(Open Graph) / Social card: SNS공유 시 썸네일, 제목, 설명 등 미리보기 박스에 나타나는 정보  
   대표 이미지가 없다면, 링크를 공유해도 썸네일이 나오지 않습니다.

  ```html
  <!-- 페이스북, 카카오 등 -->
  <meta property="og:title" content="Head 태그 마스터 가이드" />
  <meta property="og:description" content="Next.js와 HTML의 head 태그를 활용해 SEO와 OG를 최적화하는 방법을 설명합니다." />
  <meta property="og:image" content="https://yourdomain.com/images/og-head-guide.png" />
  <meta property="og:url" content="https://yourdomain.com/head-guide" />
  <meta property="og:type" content="article" />
  ```

- 리소스 관리: 브라우저가 리소스를 어떻게 받아올지 힌트를 줌

  ```html
  <-- preload - 폰트 파일 preload (렌더링 차단 요소 미리 불러오기) -->

  <link rel="preload" href="/fonts/Pretendard.woff2" as="font" type="font/woff2" crossorigin="anonymous" />

  <-- preconnect - 다른 도메인의 리소스를 미리 연결 (DNS+TLS 핸드셰이크 미리 함) - 외부 리소스 도메인 preconnect -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://cdn.jsdelivr.net" crossorigin />

  <-- dns-prefetch - DNS 조회만 먼저 해놓음 (preconnect보다 가벼움) -->
  <link rel="dns-prefetch" href="//fonts.googleapis.com" />
  ```

이외에도 보안 & 정책 메타, 다국어 등 다양한 정보를 **<head>**에 담아둘 수 있습니다.

이러한 강점들을 수치적으로 확인해볼수도 있습니다!

### 1.2 Lighthouse로 head의 강점 알아보기

[meta-description](https://developer.chrome.com/docs/lighthouse/seo/meta-description)을 확인해보면 head에 meta태그를 활용하여 seo를 높이는 방법을 설명해주고 있습니다.

> The element provides a summary of a page's content that search engines include in search results. A high-quality, unique meta description makes your page appear more relevant and can increase your search traffic.

이와 관련하여 seo에 유리하게 작성하는 법을 알려주고 있으며, meta태그가 없을 시 다양한 문구를 노출합니다. [관련 코드](https://github.com/GoogleChrome/lighthouse/blob/main/core/audits/seo/meta-description.js)

이런 경고등은 lighthouse의 SEO와 Best Practices, OG/social 에서 위 코드의 UIString에 해당하는 경고와 함께 이 페이지의 lighthouse 점수를 떨어트립니다.

이제 각 프레임워크에서 어떻게 대응을 하는지 알아보겠습니다.

## 2\. react의 csr환경에서의 <head>

### 2.1 react

따로 서버를 세팅하지않고 CSR로 운영되는 react 프로젝트의 경우, 초기렌더링 시 index.html에 들어있는 메타데이터만 즉각 노출됩니다.

즉 모든페이지에 공통으로 쓸 metadata 또는 og등만 기입할 수 있고, 페이지별, 데이터별 meta데이터를 수정할 수 없습니다. 이를 해결하기위해 react-helmet같은 라이브러리는 렌더링시점에 **<head>**를 조작하여 페이지별로 메타 데이터를 넣어주곤 하는데, 이 방법 또한 크롤러나 sns봇등이 읽지못하여 seo등을 크게 향상시키지는 못합니다.

### 2.2 react-helmet 동작원리

```react
<Helmet>
  <title>페이지 타이틀</title>
  <meta name="description" content="..." />
</Helmet>
```

1. `\<Helmet>`을 리액트 트리에 넣습니다.
2. react가 렌더링될때 helmet내부의 render()를 호출합니다.

```javascript
// HOC로 helmet컴포넌트 정의
const Helmet = Component => class HelmetWrapper extends React.Component {
      ...

      render() {
        const {children, ...props} = this.props;
        let newProps = {...props};

        if (children) {
          newProps = this.mapChildrenToProps(children, newProps);
        }

       return <Component {...newProps} />;
      }
      ...
}

// 사이드 이펙트 수행할 컴포넌트 생성
const HelmetSideEffects = withSideEffect(
    reducePropsToState,
    handleClientStateChange,
    mapStateOnServer
)(NullComponent);

// 내보내기
const HelmetExport = Helmet(HelmetSideEffects);
```

3. <Helmet>은 HelmetWrapper의 인스턴스를 생성한것이며, 렌더링 시 HelmetWrapper.render()를 호출하게 됩니다.
4. 이때 `<Component {...newProps} />;`에 react에서 주입한 데이터가 들어간 후 사용가능한 데이터로 정제 후 HelmetSideEffects가 실행됩니다.
5. handleClientStateChange의 commitTagChanges에 의해 태그가 만들어지고, 이를 reducePropsToState를 통해 조합하여 <head>데이터를 수정하게 됩니다. [상세코드](https://github.com/nfl/react-helmet/blob/master/src/HelmetUtils.js)

### 2.3 결론

결국 react-helmet을 써도 **렌더링 시점**에 head가 변경되어 ssr환경에서 html을 완성한 상태로 전해주는것에 비해 성능이 떨어집니다. 이는 ssr을 적용하면 해소가 되며, 다음은 이 아티클의 주제인 Nextjs에서 **<head>**를 다루는 방법을 소개하겠습니다.

## 3\. Next.js에서의 <head>

앞서 CSR환경의 React에서의 **<head>**의 변경은 렌더링 시점에 조작되기 때문에 SEO나 OG에 불리했습니다. 이제 SSR을 지원하는 Next.js에서의 **<head>**를 알아보겠습니다.

Next.js는 기본적으로 SSR을 지원하므로, **<head>** 데이터를 서버에서 HTML에 포함시켜 전달할 수 있습니다. 이로인해 크롤러,SNS, 검색엔진 모두 제대로 읽을 수 있습니다.

### 3.1 Next.js에서의 <head> 구성 방법

App router기준 Next.js에서 **<head>**를 구성하는 방식은 크게 두 가지로 나뉩니다. 사용 목적에 알맞게 나누어 사용하는것이 좋습니다.

| 구성 방식     | 설명                                                               | 용도                          |
| ------------- | ------------------------------------------------------------------ | ----------------------------- |
| metadata API  | SEO, title, description, OG, Twitter, favicon 등을 선언형으로 구성 | 페이지 정보 메타 구성         |
| head.tsx 파일 | 외부 폰트, 스크립트, SDK 삽입 등 <script>,<link> 구성              | 리소스 삽입, 커스텀 제어 필요 |

📎 [공식 문서: Metadata API](https://nextjs.org/docs/app/building-your-application/optimizing/metadata)  
📎 [공식 문서: head.tsx 파일 구성](https://nextjs.org/docs/app/api-reference/file-conventions/head)

우선 metadata API 부터 둘러보겠습니다.

### 3.2 metadata API

metadata는 **<script>**와 **<link>**를 제외한 모든 메타 정보를 선언형으로 정의할 수 있는 API입니다.

page나 layout에 예약된 변수명 metadata나 generateMetadata를 내보내면 SSR시 next.js가 자동으로 파싱하여 에 삽입해줍니다.

```text
클라이언트 요청
   ↓
NextServer.getRequestHandler()
   ↓
renderToHTMLOrFlight (app-render.tsx)
   ↓
createComponentTree → getMetadataComponents()
   ↓
resolveMetadata()
   ↓
resolveMetadataElements()
   ↓
MetadataTree (JSX) → <head> 렌더링 시 삽입
```

**핵심 소스 파일**

- [renderToHTMLOrFlight](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/app-render/app-render.tsx)
- [createMetadataComponents](https://github.com/vercel/next.js/blob/canary/packages/next/src/lib/metadata/create-metadata-components.tsx)
- [resolveMetadata](https://github.com/vercel/next.js/blob/canary/packages/next/src/lib/metadata/resolve-metadata.ts)
- [resolveMetadataElements](https://github.com/vercel/next.js/blob/canary/packages/next/src/lib/metadata/resolve-elements.ts)

### 3.3 head.tsx

meta api는 seo/og중심의 정보에 최적화되어있어 외부 리소스 삽입에는 적합하지 않습니다. 이는 head.tsx를 사용하면 됩니다.

```tsx
export default function Head() {
  return (
    <>
      <link rel='preconnect' href='https://fonts.googleapis.com' />
      <link href='https://fonts.googleapis.com/css2?family=Inter&display=swap' rel='stylesheet' />
    </>
  );
}
```

### 3.4 결론

next는 metadata api와 head를 분리하여, 정적 메타 데이터와 동적 메타데이터, 외부 리소스 등을 명확히 구분할 수 있도록 설계하였으며, 성능 및 유지보수 측면에서 용이합니다.# \[nextjs 톺아보기\]

## 서론

안녕하세요! 이번에 nextjs 톺아보기 스터디에 들어가게 되었는데요! 이번 주차 파트인 head를 정의부터 시작해서 react에서의 처리법, next의 처리법을 비교하며 nextjs의 head 동작원리에 다가가보겠습니다!

## 1\. 왜 는 중요한가?

는 와 다르게 컨텐츠로는 표현되지 않습니다.([What is the HTML head?](https://developer.mozilla.org/en-US/docs/Learn_web_development/Core/Structuring_content/Webpage_metadata#what_is_the_html_head)) 대신에 페이지의 metadata를 정의하는 공간입니다.

### 1.1 head가 중요한 이유

대표적으로 아래와 같습니다

- SEO(검색 엔진 최적화): 검색 엔진이 HTML을 읽고 인덱싱할 때 참고하는 메타 정보  
   검색 엔진들은 title, description, robots, canonical 등을 중요하게 반응합니다.

  ```html
  <!-- 문서 제목 -->
  <title>프론트엔드 개발자를 위한 Head 완전정복</title>

  <!-- 페이지 설명 -->
  <meta name="description" content="Next.js와 HTML의 head 태그를 심층 분석하고 SEO 최적화 방법을 소개합니다." />

  <!-- 검색 엔진에 노출 허용 -->
  <meta name="robots" content="index, follow" />

  <!-- 작성자 정보 -->
  <meta name="author" content="박동현" />

  <!-- 사이트 정규 주소 (중복 방지) -->
  <link rel="canonical" href="https://yourdomain.com/head-guide" />
  ```

- OG(Open Graph) / Social card: SNS공유 시 썸네일, 제목, 설명 등 미리보기 박스에 나타나는 정보  
   대표 이미지가 없다면, 링크를 공유해도 썸네일이 나오지 않습니다.

  ```html
  <!-- 페이스북, 카카오 등 -->
  <meta property="og:title" content="Head 태그 마스터 가이드" />
  <meta property="og:description" content="Next.js와 HTML의 head 태그를 활용해 SEO와 OG를 최적화하는 방법을 설명합니다." />
  <meta property="og:image" content="https://yourdomain.com/images/og-head-guide.png" />
  <meta property="og:url" content="https://yourdomain.com/head-guide" />
  <meta property="og:type" content="article" />
  ```

- 리소스 관리: 브라우저가 리소스를 어떻게 받아올지 힌트를 줌

  ```html
  <-- preload - 폰트 파일 preload (렌더링 차단 요소 미리 불러오기) -->

  <link rel="preload" href="/fonts/Pretendard.woff2" as="font" type="font/woff2" crossorigin="anonymous" />

  <-- preconnect - 다른 도메인의 리소스를 미리 연결 (DNS+TLS 핸드셰이크 미리 함) - 외부 리소스 도메인 preconnect -->
  <link rel="preconnect" href="https://fonts.googleapis.com" />
  <link rel="preconnect" href="https://cdn.jsdelivr.net" crossorigin />

  <-- dns-prefetch - DNS 조회만 먼저 해놓음 (preconnect보다 가벼움) -->
  <link rel="dns-prefetch" href="//fonts.googleapis.com" />
  ```

이외에도 보안 & 정책 메타, 다국어 등 다양한 정보를 **<head>**에 담아둘 수 있습니다.

이러한 강점들을 수치적으로 확인해볼수도 있습니다!

### 1.2 Lighthouse로 head의 강점 알아보기

[meta-description](https://developer.chrome.com/docs/lighthouse/seo/meta-description)을 확인해보면 head에 meta태그를 활용하여 seo를 높이는 방법을 설명해주고 있습니다.

> The element provides a summary of a page's content that search engines include in search results. A high-quality, unique meta description makes your page appear more relevant and can increase your search traffic.

이와 관련하여 seo에 유리하게 작성하는 법을 알려주고 있으며, meta태그가 없을 시 다양한 문구를 노출합니다. [관련 코드](https://github.com/GoogleChrome/lighthouse/blob/main/core/audits/seo/meta-description.js)

이런 경고등은 lighthouse의 SEO와 Best Practices, OG/social 에서 위 코드의 UIString에 해당하는 경고와 함께 이 페이지의 lighthouse 점수를 떨어트립니다.

이제 각 프레임워크에서 어떻게 대응을 하는지 알아보겠습니다.

## 2\. react의 csr환경에서의 <head>

### 2.1 react

따로 서버를 세팅하지않고 CSR로 운영되는 react 프로젝트의 경우, 초기렌더링 시 index.html에 들어있는 메타데이터만 즉각 노출됩니다.

즉 모든페이지에 공통으로 쓸 metadata 또는 og등만 기입할 수 있고, 페이지별, 데이터별 meta데이터를 수정할 수 없습니다. 이를 해결하기위해 react-helmet같은 라이브러리는 렌더링시점에 **<head>**를 조작하여 페이지별로 메타 데이터를 넣어주곤 하는데, 이 방법 또한 크롤러나 sns봇등이 읽지못하여 seo등을 크게 향상시키지는 못합니다.

### 2.2 react-helmet 동작원리

```react
<Helmet>
  <title>페이지 타이틀</title>
  <meta name="description" content="..." />
</Helmet>
```

1. `\<Helmet>`을 리액트 트리에 넣습니다.
2. react가 렌더링될때 helmet내부의 render()를 호출합니다.

```javascript
// HOC로 helmet컴포넌트 정의
const Helmet = Component => class HelmetWrapper extends React.Component {
      ...

      render() {
        const {children, ...props} = this.props;
        let newProps = {...props};

        if (children) {
          newProps = this.mapChildrenToProps(children, newProps);
        }

       return <Component {...newProps} />;
      }
      ...
}

// 사이드 이펙트 수행할 컴포넌트 생성
const HelmetSideEffects = withSideEffect(
    reducePropsToState,
    handleClientStateChange,
    mapStateOnServer
)(NullComponent);

// 내보내기
const HelmetExport = Helmet(HelmetSideEffects);
```

3. <Helmet>은 HelmetWrapper의 인스턴스를 생성한것이며, 렌더링 시 HelmetWrapper.render()를 호출하게 됩니다.
4. 이때 `<Component {...newProps} />;`에 react에서 주입한 데이터가 들어간 후 사용가능한 데이터로 정제 후 HelmetSideEffects가 실행됩니다.
5. handleClientStateChange의 commitTagChanges에 의해 태그가 만들어지고, 이를 reducePropsToState를 통해 조합하여 <head>데이터를 수정하게 됩니다. [상세코드](https://github.com/nfl/react-helmet/blob/master/src/HelmetUtils.js)

### 2.3 결론

결국 react-helmet을 써도 **렌더링 시점**에 head가 변경되어 ssr환경에서 html을 완성한 상태로 전해주는것에 비해 성능이 떨어집니다. 이는 ssr을 적용하면 해소가 되며, 다음은 이 아티클의 주제인 Nextjs에서 **<head>**를 다루는 방법을 소개하겠습니다.

## 3\. Next.js에서의 <head>

앞서 CSR환경의 React에서의 **<head>**의 변경은 렌더링 시점에 조작되기 때문에 SEO나 OG에 불리했습니다. 이제 SSR을 지원하는 Next.js에서의 **<head>**를 알아보겠습니다.

Next.js는 기본적으로 SSR을 지원하므로, **<head>** 데이터를 서버에서 HTML에 포함시켜 전달할 수 있습니다. 이로인해 크롤러,SNS, 검색엔진 모두 제대로 읽을 수 있습니다.

### 3.1 Next.js에서의 <head> 구성 방법

App router기준 Next.js에서 **<head>**를 구성하는 방식은 크게 두 가지로 나뉩니다. 사용 목적에 알맞게 나누어 사용하는것이 좋습니다.

| 구성 방식     | 설명                                                               | 용도                          |
| ------------- | ------------------------------------------------------------------ | ----------------------------- |
| metadata API  | SEO, title, description, OG, Twitter, favicon 등을 선언형으로 구성 | 페이지 정보 메타 구성         |
| head.tsx 파일 | 외부 폰트, 스크립트, SDK 삽입 등 <script>,<link> 구성              | 리소스 삽입, 커스텀 제어 필요 |

📎 [공식 문서: Metadata API](https://nextjs.org/docs/app/building-your-application/optimizing/metadata)  
📎 [공식 문서: head.tsx 파일 구성](https://nextjs.org/docs/app/api-reference/file-conventions/head)

우선 metadata API 부터 둘러보겠습니다.

### 3.2 metadata API

metadata는 **<script>**와 **<link>**를 제외한 모든 메타 정보를 선언형으로 정의할 수 있는 API입니다.

page나 layout에 예약된 변수명 metadata나 generateMetadata를 내보내면 SSR시 next.js가 자동으로 파싱하여 에 삽입해줍니다.

```text
클라이언트 요청
   ↓
NextServer.getRequestHandler()
   ↓
renderToHTMLOrFlight (app-render.tsx)
   ↓
createComponentTree → getMetadataComponents()
   ↓
resolveMetadata()
   ↓
resolveMetadataElements()
   ↓
MetadataTree (JSX) → <head> 렌더링 시 삽입
```

**핵심 소스 파일**

- [renderToHTMLOrFlight](https://github.com/vercel/next.js/blob/canary/packages/next/src/server/app-render/app-render.tsx)
- [createMetadataComponents](https://github.com/vercel/next.js/blob/canary/packages/next/src/lib/metadata/create-metadata-components.tsx)
- [resolveMetadata](https://github.com/vercel/next.js/blob/canary/packages/next/src/lib/metadata/resolve-metadata.ts)
- [resolveMetadataElements](https://github.com/vercel/next.js/blob/canary/packages/next/src/lib/metadata/resolve-elements.ts)

### 3.3 head.tsx

meta api는 seo/og중심의 정보에 최적화되어있어 외부 리소스 삽입에는 적합하지 않습니다. 이는 head.tsx를 사용하면 됩니다.

```tsx
export default function Head() {
  return (
    <>
      <link rel='preconnect' href='https://fonts.googleapis.com' />
      <link href='https://fonts.googleapis.com/css2?family=Inter&display=swap' rel='stylesheet' />
    </>
  );
}
```

### 3.4 결론

next는 metadata api와 head를 분리하여, 정적 메타 데이터와 동적 메타데이터, 외부 리소스 등을 명확히 구분할 수 있도록 설계하였으며, 성능 및 유지보수 측면에서 용이합니다.
