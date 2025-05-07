# w06. dynamic

- 세 가지 옵션 지원
    - `dynamic(import('../hello-world'))`
    - `dynamic(() => import('../hello-world'))`
    - `dynamic({loader: import('../hello-world')})`
- react-loadable 쓰는 듯?

```tsx
// 이런 dynamic은
const ComponentA = dynamic(() => import('../components/A'))

// 결굴 이런 형태가 된다.
const ComponentA = Loadable({
  loader: () => import('../components/A'),
});
```

- react-loadable 에서는 flash of loading component를 피하기 위해 기본 200ms의 딜레이가 있고, 그 딜레이가 지나면 `pastDelay`가 true가 된다.

```ts
function Loading(props) {
  if (props.error) {
    return <div>Error! <button onClick={ props.retry }>Retry</button></div>;
  } else if (props.pastDelay) {
    return <div>Loading...</div>;
  } else {
    return null;
  }
}
```

```ts
// loadable.shared-runtime.tsx

// state는 useSyncExternalStore로 싱크한다.
// 로딩이 완료되지 않은 상태면 loading 컴포넌트를 렌더링하고
// 로딩이 완료되면 loaded 컴포넌트를 렌더링한다.
return React.useMemo(() => {
  if (state.loading || state.error) {
    return React.createElement(opts.loading, {
      isLoading: state.loading,
      pastDelay: state.pastDelay,
      timedOut: state.timedOut,
      error: state.error,
      retry: subscription.retry,
    })
  } else if (state.loaded) {
    return React.createElement(resolve(state.loaded), props)
  } else {
    return null
  }
}, [props, state])
```
