---
title: "TanStack Queryのpersistersで別タブで開いた時にキャッシュを共有する"
emoji: "🏝️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tanstackquery", "キャッシュ", "react"]
publication_name: "galapagos"
published: false
---

この記事は[株式会社ガラパゴス（有志）アドベントカレンダー 2025](https://qiita.com/advent-calendar/2025/galapagos)の 9 日目の記事です。

## TanStack Query のキャッシュをタブ間で共有する動機

CSR（クライアントサイドレンダリング）オンリーの Web アプリケーションで、Restful API を叩くためのクライアントライブラリとして、TanStack Query を使用しています。いわゆる SPA（Single Page Application）の開発なのですが、ブラウザ的にはよく使う機能だけど、TanStack Query との組み合わせでパフォーマンス面の UX が気になるパターンがあります。

それが、**「別タブ（別ウィンドウ）で開く / リロード」（ハードナビゲーション）** です。

なぜかというと、TanStack Query のキャッシュはライブラリ内部のインメモリ `QueryCache`（JS オブジェクト/Map）なので、別タブやリロードでは引き継がれず、初期化されます。

https://github.com/TanStack/query/blob/f15b7fcc01e995ab8835f1b1cc82ebb472c1ff64/packages/query-core/src/queryCache.ts#L92-L98

画面遷移や UI 更新で変更頻度が少ないデータ、例えばユーザー情報など、`staleTime` や `gcTime` を長く設定してキャッシュでの維持を頑張っても、リロードや別タブで開くとキャッシュは失われてしまいます。

SPA 開発して一番つらみを感じたのは、`prefetchQuery`で遷移先のデータを事前取得することが、遷移先がハードナビゲーションだと全く意味をなさないことです。

### `broadcastQueryClient`は？

最初は、現在 Experimental の`broadcastQueryClient`で行けるのかな〜とぼんやりと思っていたのですが、
`broadcastQueryClient`は既に開いているタブ間でキャッシュを共有するためのもので、新規タブ・ウィンドウは別々の`QueryCache`を持ちます。

https://github.com/TanStack/query/issues/2142

## `Persister`が必要

調べた限り、ハードナビゲーションでキャッシュを維持するには、`Persister`を使うしかなさそうです。

`Persister`は、`QueryClient`を`localStorage`や`IndexedDB`に保存して永続化するためのユーティリティ群で、
TanStack Query v5 時点の公式ドキュメントでは下記の型になっています。

```ts
export interface Persister {
  persistClient(persistClient: PersistedClient): Promisable<void>;
  restoreClient(): Promisable<PersistedClient | undefined>;
  removeClient(): Promisable<void>;
}
```

https://tanstack.com/query/latest/docs/framework/react/plugins/persistQueryClient

今回は、公式ドキュメントの`IndexedDB`の実装をそのまま参考に、`prefetchQuery`を使って別タブで開く際に、永続化されたキャッシュを利用して表示速度の体験を改善する実装を試みました。

## `Persister`の実装

下記は公式ドキュメントの`IndexedDB`の実装をそのまま転載したものです。

```tsx
import { get, set, del } from "idb-keyval";
import {
  PersistedClient,
  Persister,
} from "@tanstack/react-query-persist-client";

/**
 * Creates an Indexed DB persister
 * @see https://developer.mozilla.org/en-US/docs/Web/API/IndexedDB_API
 */
export function createIDBPersister(idbValidKey: IDBValidKey = "reactQuery") {
  return {
    persistClient: async (client: PersistedClient) => {
      await set(idbValidKey, client);
    },
    restoreClient: async () => {
      return await get<PersistedClient>(idbValidKey);
    },
    removeClient: async () => {
      await del(idbValidKey);
    },
  } satisfies Persister;
}
```

上記の`createIDBPersister`を使うには、今まで`QueryClientProvider`を渡していたアプリのなるべく上の階層で、`QueryClientProvider`を`PersistQueryClientProvider`に置き換えます。

```tsx
import { PersistQueryClientProvider } from "@tanstack/react-query-persist-client";

const persister = createIDBPersister();

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 1000 * 60 * 60 * 24, // 24 hours
    },
  },
});

export function App() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{ persister }}
    >
      <AppComponent />
    </PersistQueryClientProvider>
  );
}
```

これで、Chrome の devtools の Application タブの IndexedDB で永続化されたキャッシュを確認できます。

![](https://storage.googleapis.com/zenn-user-upload/04a11ee7382b-20251208.png)

### prefetchQuery を使った別タブでの表示速度の改善

`Persister`が実装できたので、あとは`prefetchQuery`を使って遷移先のデータを事前取得することで、別タブで開く時に永続化されたキャッシュからデータを取得できます。そうすることで、開いた別タブでは永続化キャッシュの同一`queryKey`のデータを利用するので、フェッチを経ずに表示できます。

下記は公式の`prefetchQuery`の例を少し変更して、React Router `Link`コンポーネントの`onMouseEnter`と`onFocus`で`prefetchQuery`を呼び出すものです。

```jsx
function ShowDetailsButton() {
  const queryClient = useQueryClient();

  const prefetch = () => {
    queryClient.prefetchQuery({
      queryKey: ["details"],
      queryFn: getDetailsData,
      gcTime: 1000 * 60 * 60 * 24, // 24 hours
    });
  };

  return (
    <Link
      to="/details"
      target="_blank"
      onMouseEnter={prefetch}
      onFocus={prefetch}
    >
      To Details Page
    </Link>
  );
}
```

https://tanstack.com/query/latest/docs/framework/react/guides/prefetching#prefetch-in-event-handlers

### persisters の保持期間の設定に注意

https://tanstack.com/query/latest/docs/framework/react/plugins/persistQueryClient#how-it-works

上記のドキュメントにも書いてあるのですが、キャッシュする時間の長さは、`maxAge`というオプションで設定することができ、v5 の時点ではデフォルトが 24 時間（1000 \* 60 \* 60 \* 24 ミリ秒）になっています。

```tsx
<PersistQueryClientProvider
  client={queryClient}
  persistOptions={{
    persister: persister,
    maxAge: 1000 * 60 * 60 * 24, // default: 24 hours,
  }}
>
```

気をつけなければいけないのが、クエリの`gcTime`が`persistOptions`の`maxAge`よりも短いと、永続化キャッシュが意図した時間よりも早くクリアされてしまうことです。`gcTime`のデフォルト値は 5 分のため、`maxAge`を長くしても表示に使われなかったデータは、`gcTime`で破棄されてしまいます。

下記のように`gcTime`を`maxAge`に合わせるか、`Infinity`に設定することがドキュメントで言及されています。

```jsx
const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      gcTime: 1000 * 60 * 60 * 24, // 24 hours
      // or
      // gcTime: Infinity,
    },
  },
});
```

## クエリごとにキャッシュするかを決めたい

実はここが本題だったりもします。

すべての API を永続化したいのであれば、紹介したドキュメントの実装をそのまま使えば良いのですが、実際は永続化したいもの、そうでないものがあるパターンがあるかと思います。

思いつく方法としては、永続化したくないクエリの`gcTime`を短くする、ということもあるのですが、実際に実装したのは、**デフォルトでは永続化したくない** というパターンでした。

そのためには、下記のように`dehydrateOptions.shouldDehydrateQuery`を設定する必要があります。

```jsx
<PersistQueryClientProvider
  client={queryClient}
  persistOptions={{
    persister: persister,
    maxAge: 1000 * 60 * 60 * 24,
    dehydrateOptions: {
      shouldDehydrateQuery: (query) =>
        Boolean(query.meta?.persist === true),
    },
  }}
>
```

`dehydrateOptions.shouldDehydrateQuery`は`(query: Query) => boolean`という型になっていて、`true`を返すとそのクエリが永続化に含まれ、`false`を返すと除外されます。

詳細は下記の issue で紹介されています。

https://github.com/TanStack/query/discussions/3568

ですので、上記を踏まえて、特定のクエリだけ永続化したい場合は下記のように実装します。

```jsx
import { PersistQueryClientProvider } from "@tanstack/react-query-persist-client";
import { useQuery, useQueryClient } from "@tanstack/react-query";

const persister = createIDBPersister();

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      // デフォルトを使うか、短い時間を設定
      gcTime: 1000 * 60 * 5,
    },
  },
});

export function App() {
  return (
    <PersistQueryClientProvider
      client={queryClient}
      persistOptions={{
        persister: persister,
        maxAge: 1000 * 60 * 60 * 24,
        dehydrateOptions: {
          shouldDehydrateQuery: (query) =>
            Boolean(query.meta?.persist === true),
        },
      }}
    >
      <NeedRealTimeDataComp />
      <ShowDetailsButton />
    </PersistQueryClientProvider>
  );
}

function NeedRealTimeDataComp() {
  // このクエリは永続化しない（IndexedDB には保存されない）
  const { data } = useQuery({
    queryKey: ["real-time-data"],
    queryFn: getRealTimeData,
    // shouldDehydrateQuery が persist === true のみを対象としているため、meta の指定は省略しても同じ動作
    meta: {
      persist: false,
    },
  });

  return (
    <div>
      <p>Real-time data: {data}</p>
    </div>
  );
}

function ShowDetailsButton() {
  const queryClient = useQueryClient();

  const prefetch = () => {
    queryClient.prefetchQuery({
      queryKey: ["details"],
      queryFn: getDetailsData,
      gcTime: 1000 * 60 * 60 * 24, // クエリごとにgcTimeは長く設定する
      meta: {
        persist: true,
      },
    });
  };

  return (
    <Link
      to="/details"
      target="_blank"
      onMouseEnter={prefetch}
      onFocus={prefetch}
    >
      To Details Page
    </Link>
  );
}
```

## 余談: 永続化キャッシュをいつクリアするか

キャッシュを永続化するということは、どのタイミングでキャッシュをクリアするか、ということもセットで考える必要があります。実際の実装では`gcTime`と`maxAge`を組み合わせだけで間に合ったのですが、ビルドなどの変更を検知してキャッシュをクリアしたい場合などは、`buster`オプションが利用できるようです。実際に使っていないため記事には書けないですが、永続化をもっとフル活用する場合には必要になりそうです。

```ts
persistQueryClient({ queryClient, persister, buster: buildHash });
persistQueryClientSave({ queryClient, persister, buster: buildHash });
persistQueryClientRestore({ queryClient, persister, buster: buildHash });
```

https://tanstack.com/query/latest/docs/framework/react/plugins/persistQueryClient#cache-busting
