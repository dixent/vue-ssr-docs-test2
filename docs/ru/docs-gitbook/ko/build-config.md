# 빌드 설정

클라이언트 전용 프로젝트 를 webpack 을 구성 하는 방법 을 이미 있다고 가정 합니다. ССР 프로젝트 설정 은 거의 유사 하지만 구성 을 세가지 로 *(기본,* **클라이언트 와 서버)** 로 나누는 것이 좋습니다. 기본 구성 에는 출력 될 경로 별칭 및 로더 와 같은 두 되는 설정 들이 있습니다. 와 클라이언트 설정 은 webpack [-merge](https://github.com/survivejs/webpack-merge) 를 사용해 기본 설정 을 확장 합니다.

## 서버 설정

설정 은 `createBundleRenderer` 에 전달 될 서버 번들 을 생성 하기 위한 것 입니다. 아래와 같습니다.

```js
const merge = require('webpack-merge')
const nodeExternals = require('webpack-node-externals')
const baseConfig = require('./webpack.base.config.js')
const VueSSRServerPlugin = require('vue-server-renderer/server-plugin')
module.exports = merge(baseConfig, {
  // Point entry to your app's server entry file
  entry: '/path/to/entry-server.js',
  // This allows webpack to handle dynamic imports in a Node-appropriate
  // fashion, and also tells `vue-loader` to emit server-oriented code when
  // compiling Vue components.
  target: 'node',
  // For bundle renderer source map support
  devtool: 'source-map',
  // This tells the server bundle to use Node-style exports
  output: {
    libraryTarget: 'commonjs2'
  },
  // https://webpack.js.org/configuration/externals/#function
  // https://github.com/liady/webpack-node-externals
  // Externalize app dependencies. This makes the server build much faster
  // and generates a smaller bundle file.
  externals: nodeExternals({
    // do not externalize dependencies that need to be processed by webpack.
    // you can add more file types here e.g. raw *.vue files
    // you should also whitelist deps that modifies `global` (e.g. polyfills)
    whitelist: /\.css$/
  }),
  // This is the plugin that turns the entire output of the server build
  // into a single JSON file. The default file name will be
  // `vue-ssr-server-bundle.json`
  plugins: [
    new VueSSRServerPlugin()
  ]
})
```

`vue-ssr-server-bundle.json` 이 생성 된 후 파일 경로 를 `createBundleRenderer` 에 전달 합니다.

```js
const { createBundleRenderer } = require('vue-server-renderer')
const renderer = createBundleRenderer('/path/to/vue-ssr-server-bundle.json', {
  // ...other renderer options
})
```

, 번들 을 객체 로 만들어 `createBundleRenderer` 에 전달할 수 있습니다. 이는 개발 중 핫 리로드 를 사용할 때 유용 합니다. HackerNews 의 데모 [설정](https://github.com/vuejs/vue-hackernews-2.0/blob/master/build/setup-dev-server.js) 을 참조 하세요.

### Внешний вид 주의 사항

`externals` элементы 옵션 에서는 CSS 파일 을 허용 하는 목록 에 추가 합니다. 의존성 에서 가져온 CSS 는 여전히 webpack 에서 처리 해야 합니다. webpack 을 사용 하는 다른 유형 의 파일 을 가져 오는 경우 (예: `*.vue` , `*.sass` ) 파일 을 허용 목록 에 추가 해야 합니다.

`runInNewContext: 'once'` 또는 `runInNewContext: true` 를 사용 하는 경우 `전역 변수` 를 수정 하는 폴리 필 (Polyfill) 을 허용 목록 에 추가 해야 합니다. 예 를 들어 `babel-polyfill` 이 있습니다. 새 컨텍스트 모드 를 사용할 때 **서버 번들 의 내부 코드 가 으로 `전역` 객체 를 때문 입니다.** Узел 7.6 이상 을 사용할 때 서버 에서는 실제로 필요 하지 않으므로 클라이언트 에서 가져 오는 것이 더 쉽습니다.

## 클라이언트 설정

클라이언트 설정 은 기본 설정 과 거의 동일 합니다. 클라이언트 의 `entry` 파일 을 가리키면 됩니다. 그 외에도 `CommonsChunkPlugin` 을 사용 하는 경우 서버 번들 에 단일 запись 청크 가 필요 하기 때문에 클라이언트 설정 에서만 사용해야 합니다 .

### `clientManifest` 생성

> 2.3.0 버전 이후 지원

서버 번들 외에도 클라이언트 빌드 매니페스트 를 생성 할 수도 있습니다. 클라이언트 매니페스트 와 서버 번들 을 사용 서버 빌드 정보 가 *모두* 포함 되므로 [프리 로드 / 프리 페치 디렉티브](https://css-tricks.com/prefetching-preloading-prebrowsing/) 와 CSS 링크 / скрипт 태그 를 렌더링 된 HTML 에 자동 으로 삽입 할 수 있습니다.

두가지 장점 이 있습니다.

1. 된 파일 이름 에 해시 가 있을 올바른 URL 을 삽입 하기 위해 `html-webpack-plugin` 을 대체 할 수 있습니다.
2. webpack 의 주문형 코드 분할 기능 을 활용 활용 을 렌더링 할 때 최적 의 청크 를 프리 로드 / 프리 페치 하고 클라이언트 에 폭포수 요청 을 피하기 위해 필요한 에 `<script></script>` 태그 를 지능적 으로 삽입 할 수 있습니다 . 이는 TTI (첫 작동 까지 의 시간) 을 개선 합니다.

클라이언트 매니페스트 를 사용 하려면 클라이언트 설정 은 아래와 같아야 합니다.

```js
const webpack = require('webpack')
const merge = require('webpack-merge')
const baseConfig = require('./webpack.base.config.js')
const VueSSRClientPlugin = require('vue-server-renderer/client-plugin')
module.exports = merge(baseConfig, {
  entry: '/path/to/entry-client.js',
  plugins: [
    // Important: this splits the webpack runtime into a leading chunk
    // so that async chunks can be injected right after it.
    // this also enables better caching for your app/vendor code.
    new webpack.optimize.CommonsChunkPlugin({
      name: "manifest",
      minChunks: Infinity
    }),
    // This plugins generates `vue-ssr-client-manifest.json` in the
    // output directory.
    new VueSSRClientPlugin()
  ]
})
```

그런 다음 생성 한 클라이언트 매니페스트 를 페이지 템플릿 과 함께 사용 합니다.

```js
const { createBundleRenderer } = require('vue-server-renderer')
const template = require('fs').readFileSync('/path/to/template.html', 'utf-8')
const serverBundle = require('/path/to/vue-ssr-server-bundle.json')
const clientManifest = require('/path/to/vue-ssr-client-manifest.json')
const renderer = createBundleRenderer(serverBundle, {
  template,
  clientManifest
})
```

이 설정 을 사용 하면 코드 분할 을 서버 렌더링 된 HTML 이 다음 과 같이 표시 됩니다. (전체 가 자동 으로 주입 됩니다)

```html
<html>
  <head>
    <!-- chunks used for this render will be preloaded -->
    <link rel="preload" href="/manifest.js" as="script">
    <link rel="preload" href="/main.js" as="script">
    <link rel="preload" href="/0.js" as="script">
    <!-- unused async chunks will be prefetched (lower priority) -->
    <link rel="prefetch" href="/1.js" as="script">


    <!-- app content -->
    <div data-server-rendered="true"><div data-segment-id="430777">async </div></div>
    <!-- manifest chunk should be first -->
    <script src="/manifest.js"></script>
    <!-- async chunks injected before main chunk -->
    <script src="/0.js"></script>
    <script src="/main.js"></script>
  </body>
</html>
```

### 수동 에셋 주입

`template` 렌더링 옵션 을 제공 하면 기본적 으로 에셋 주입 이 자동 으로 수행 됩니다. 그러나 템플릿 에 에셋 을 삽입 하는 방법 세밀 하게 제어 템플릿 을 전혀 사용 하지 않을 수도 있습니다. 이 경우 렌더러 를 만들 때 inject `inject: false` 를 전달 하고 수동 으로 에셋 주입 을 할 수 있습니다.

`renderToString` 콜백 에서 전달한 `context` 객체 는 다음 메소드 를 노출 합니다.

- `context.renderStyles()`

렌더링 중에 사용 된 `*.vue` 컴포넌트 에서 수집 된 모든 CSS 가 포함 된 인라인 `<style></style>` 태그 가 반환 됩니다. 자세한 내용 은 [CSS 관리](./css.md) 를 참조 하십시오.

`clientManifest` 가 제공 되면 반환 되는 문자열 webpack 에서 생성 한 CSS 파일 (예: `extract-text-webpack-plugin` 또는 `file-loader` 로 추가 된) 에 대한 `<link rel="stylesheet">` 태그 가 포함 됩니다.

- `context.renderState(options?: Object)`

이 메소드 는 `context.state` 를 직렬화 하고 состояние (상태) 를 `window.__INITIAL_STATE__` 로 포함 하는 인라인 스크립트 를 리턴 합니다.

컨텍스트 состояние (상태) 키 와 윈도우 состояние (상태) 키 는 옵션 객체 를 전달 하여 할 수 있습니다.

```js
  context.renderState({
    contextKey: 'myCustomState',
    windowKey: '__MY_STATE__'
  })
  // -> <script>window.__MY_STATE__={...}</script>
```

- `context.renderScripts()`
    - `clientManifest` 를 필요 로 합니다.

이 메소드 는 클라이언트 애플리케이션 이 시작 하는데 필요한 `<script></script>` 태그 를 반환 합니다. 애플리케이션 코드 에서 비동기 코드 분할 을 사용 경우 이 포함 할 올바른 비동기 청크 를 지능적 으로 유추 합니다.

- `context.renderResourceHints()`
    - `clientManifest` 를 필요 로 합니다.

이 메소드 는 현재 렌더링 된 페이지 에 필요한 `<link rel="preload/prefetch">` 리소스 힌트 를 반환 합니다. 기본적 으로 다음 과 같습니다.

- 페이지 에 필요한 JavaScript 및 CSS 파일 을 미리 로드
- 나중에 필요할 수 있는 비동기 JavaScript 청크 프리 페치

미리 로드 된 파일 은 [`shouldPreload`](./api.md#shouldpreload) 옵션 을 사용해 추가 로 사용자 정의 할 수 있습니다.

- `context.getPreloadFiles()`
    - `clientManifest` 를 필요 로 합니다.

이 메소드 는 문자열 을 반환 하지 미리 로드 해야 할 에셋 을 나타내는 파일 객체 의 배열 을 반환 합니다. 이는 프로그래밍 방식 으로 HTTP / 2 서버 푸시 를 하는데 사용할 수 있습니다.

`createBundleRenderer` 에 전달 된 `template` 은 `context` 를 사용 하여 보간 되므로 템플릿 안에서 이러한 메소드 를 사용할 수 있습니다. (inject `inject: false` 옵션 과 함께)

```html


    <!-- use triple mustache for non-HTML-escaped interpolation -->
    {{{ renderResourceHints() }}}
    {{{ renderStyles() }}}


    <!--vue-ssr-outlet-->
    {{{ renderState() }}}
    {{{ renderScripts() }}}


```

`template` 을 사용 하지 않으면 문자열 을 직접 연결할 수 있습니다.
