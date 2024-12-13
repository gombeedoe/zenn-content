---
title: "react-router(v6)のLazy Loadingでページ遷移時のdynamically importedエラーを回避する"
emoji: "🔗"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react", "reactrouter", "vite"]
publication_name: "galapagos"
published: false
---

この記事は[株式会社ガラパゴス（有志）アドベントカレンダー2024](https://qiita.com/advent-calendar/2024/galapagos)の14日目の記事です。

react-router v7安定版がリリースされましたね！🎉
ということで、react-router **v6**について記事を書こうと思います。

## v6からv7への移行と、この記事の注意点

v6でdata router(`createBrowserRouter`と`RouterProvider`)を使った場合に、route定義で`lazy`オプションが使えました。この記事では、v6でdata routerで`lazy`を使っている場合の実装を記載しています。

v7でReact Router Viteプラグインを使った場合、[`lazy`は`file`というオプションに置き換える必要があります。](https://reactrouter.com/upgrading/router-provider#6-migrate-your-routes)`file`は`string`型のみ受け付けるため、`file`ではこの記事で紹介する方法は使えなくなると思います。

v7.0.2時点での型は下記になっています。
```ts: remix-run/react-router/packages/react-router-dev/config/routes.ts
interface RouteConfigEntry {
    /**
     * The unique id for this route.
     */
    id?: string;
    /**
     * The path this route uses to match on the URL pathname.
     */
    path?: string;
    /**
     * Should be `true` if it is an index route. This disallows child routes.
     */
    index?: boolean;
    /**
     * Should be `true` if the `path` is case-sensitive. Defaults to `false`.
     */
    caseSensitive?: boolean;
    /**
     * The path to the entry point for this route, relative to
     * `config.appDirectory`.
     */
    file: string;
    /**
     * The child routes.
     */
    children?: RouteConfigEntry[];
}
```

https://github.com/remix-run/react-router/blob/821ae3fc6ae6a6ac9d752d2e5f88218547beca95/packages/react-router-dev/config/routes.ts#L68-L99

React Router Viteプラグイン（フレームワークとしてv7）を使った場合、自動でcode-splittingされるのでlazyが不要なのは理解できたのですが、今回紹介する`Failed to fetch dynamically imported module`についての課題をv7がフレームワーク(React Router Viteプラグイン)としてどのように解決しているのか、そして今回の実装をそのまま移行できるのかは未検証です。

ただ、`createBrowserRouter`と`RouterProvider`を廃止しているわけではなく、『Custom Framework』という項目で使っているので、この記事が役立つことを祈っています。

https://reactrouter.com/start/framework/custom

# `Failed to fetch dynamically imported module`の問題

前置きが長くなりましたが、本題に入ります。
この実装は以下の前提で行っています。

- クライアントサイドレンダリングSPAの開発である
- react-router v6.28.0
- `createBrowserRouter`で`lazy`を使ってルート単位でコード分割している

ユーザーがWebアプリを開いた状態で、ビルド&デプロイ実行され、ユーザーがナビゲーションを行うと、ブラウザは`Failed to fetch dynamically imported module`エラーになります。

- ビルド前
  - `chunk_a_1.js`が`chunk_a_2.js`を参照
  - ブラウザは`chunk_a_1.js`を開いている
- ビルド＆デプロイ後
  - `chunk_a_1.js`は消え、`chunk_b_1.js`として生まれ変わる
  - `chunk_a_2.js`は消え、`chunk_b_2.js`として生まれ変わる
  - ブラウザは`chunk_a_1.js`を開いている
  - ブラウザがナビゲーションしようとして`chunk_a_2.js`をリクエストする
  - もうそこに`chunk_a_2.js`はいない

下記のissueに解決方法があり、これで良いかなと思いましたが課題が残りました。

```ts
lazy: () => import("./routes/Demo/Demo").catch(() => window.location.reload()),
```

https://github.com/remix-run/react-router/discussions/10333

上記の実装だと、ページAからページBへのナビゲーションで失敗した場合、**ページAでリロード**されるため、ユーザーとしては不自然な挙動だと感じました。

### 期待する挙動と実装

- エラーがない場合はソフトナビゲーション(`window.history.pushState()`)
- エラーの場合はハードナビゲーション（ページ全体がリロードされる）

挙動的には上記になってほしいのですが、`.catch(() => window.location.reload())`では実現できなかったため、下記のように実装しました。

```tsx
import * as React from "react";
import { type RouteObject, createBrowserRouter } from "react-router-dom";

type ImportError = {
  path: string;
  error: unknown;
};

// リロードするだけのコンポーネントを定義
const LocationReload = () => {
  React.useEffect(() => {
    window.location.reload();
  }, []);
  return null;
};

// dynamic importのcatchに渡す関数
const handleDynamicImportError = ({ path, error }: ImportError) => {

  // ネットワークが繋がっていない場合など、chunk変更以外のエラーの場合の無限リロードを防止する
  const storageKey = `react_router_reload:${path}`;
  if (!sessionStorage.getItem(storageKey)) {
    sessionStorage.setItem(storageKey, "1");

    // エラーの場合はリロードするコンポーネントを返す
    return {
      Component: LocationReload,
    } satisfies RouteObject;
  }

  throw error;
};

const router = createBrowserRouter([
  {
    path: "/",
    ErrorBoundary: () => {
      return (
        <>
          <h1>Uh oh!</h1>
          <p>Something went wrong!</p>
          <button onClick={() => window.location.reload()}>
            Click here to reload the page
          </button>
        </>
      );
    },
    lazy: () =>
      import("./routes/Demo/Demo").catch((error) =>
        handleDynamicImportError({
          path: "/",
          error,
        })
      ),
  },
]);
```

無限ループ防止のための`sessionStorage`実装は[@tanstack/react-router](https://tanstack.com/router/latest)を参考にしました。

https://github.com/TanStack/router/blob/ccf79726266fe0031bf34be52c7d2e1caaf7acf2/packages/react-router/src/lazyRouteComponent.tsx#L84-L96

## 結び

Viteには`vite:preloadError`イベントが用意されていて、これを使って上手く実装できないかとも思ったのですが、
やはり遷移前のページでリロードされるため、今回の実装に落ち着きました。

```ts
window.addEventListener('vite:preloadError', (event) => {
  window.location.reload() // for example, refresh the page
})
```

https://vite.dev/guide/build#load-error-handling

v7のReact Router Viteプラグインではこのイベントを使っていたりするのでしょうか。
願わくば、v7へのアップデートで今回の実装が不要になるといいなぁと思っています。
