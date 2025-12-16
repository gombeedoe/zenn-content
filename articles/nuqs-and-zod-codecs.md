---
title: "nuqsとZod codecsを使ってURLSearchParamsを快適にしたい"
emoji: "🦆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nuqs", "zod", "react", "reactrouter"]
publication_name: "galapagos"
published: true
---

この記事は[株式会社ガラパゴス（有志）アドベントカレンダー 2025](https://qiita.com/advent-calendar/2025/galapagos)の 16 日目の記事です。

## URLSearchParams を使った React の状態管理の悩みどころ

URLSearchParams を使った React 状態管理は URL 共有やブラウザリロードで状態を復元できたりと便利ですが、`URLSearchParams.get(name: string): string | null`と、値が文字列(見つからない場合は`null`)なので、数値やオブジェクトを扱うのが難しいです。

TanStack Router は型安全な URLSearchParams を扱える API を提供していますが、React Router の[`useSearchParams`](https://reactrouter.com/api/hooks/useSearchParams)は、`URLSearchParams`の型をそのまま返すので、バニラ JS と同じ課題があります。

そこで、各所で紹介されていますが[`nuqs`](https://nuqs.dev/)というライブラリがとても便利です。公式サイトのトップページに下記のように紹介されています。

> Type-safe search params state manager for React

元々は`Next-UseQueryState`という名前のライブラリで、名前からわかるように、Next.js 用のライブラリだったようです。

私は[`nuqs@2.0.0`](https://nuqs.dev/blog/nuqs-2)で React Router などがサポートされた段階で知って、React Router で使えるのであれば試しに使ってみようとインストールしてみたのですが、もう無い頃には戻れない感じがしています。

`nuqs`自体の解説は、採用するモチベーションも含め、とても分かりやすい動画が公開されておりますので、丸投げですが、下記がとても参考になります。

https://youtu.be/NYJgymLON2Q?si=Rjm2z2DKNO3NmqX1

## nuqs と Zod codecs

フロントのフォームバリデーションなど随所で zod を使う機会が多いため、アップデートを追っていたのですが、`zod@4.1`から[`Codecs`](https://zod.dev/codecs)という機能が追加されました。

ドキュメントからの引用ですが、下記のコードがわかりやすいかと思います。
`z.codec()`の第一引数には入力値、つまりは parse する時にどのような値が渡されるかを定義します。
第二引数には、出力値、つまりは parse された後の値のスキーマを定義します。

第三引数には、それぞれの値を変換する関数を定義します。`decode`に渡す関数の第一引数は、`codec`の第一引数、返り値は`codec`の第二引数、`encode`に渡す関数の第一引数は、`codec`の第二引数、返り値は`codec`の第一引数と同じ型でなければなりません。

```ts
const stringToDate = z.codec(
  z.iso.datetime(), // input schema: ISO date string
  z.date(), // output schema: Date object
  {
    decode: (isoString) => new Date(isoString), // ISO string → Date
    encode: (date) => date.toISOString(), // Date → ISO string
  }
);

stringToDate.decode("2024-01-15T10:30:00.000Z");
// => Date

stringToDate.encode(new Date("2024-01-15T10:30:00.000Z"));
// => string
```

`stringToDate.decode()`は文字列以外渡すと型エラーとなり、`stringToDate.encode()`は`Date`以外渡すと型エラーとなります。一方、`stringToDate.parse()`をすることも可能で、`.parse()`は何を渡しても型エラーにはならず、スキーマに合わない場合は実行時エラーになります。

`z.codec`の説明が長くなってしまいましたが、`z.codec`が提供されてしばらく経った頃だと思うのですが、[`nuqs`のドキュメンに`z.codec`を使った例](https://nuqs.dev/docs/parsers/community/zod-codecs)が掲載されました。

正直、結構難解に思いました。
ですので、この機会に自分で書いてみて理解を深めてみたいと思いました。

https://nuqs.dev/docs/parsers/community/zod-codecs

## Zod codecs を使って nuqs の parser を作る

まずは、先のドキュメントから、下記のコードを拝借します。
`parse(query) { return codec.parse(query) }`が下記のエラーになってしまったため、`safeParse`で `null`を返せるように変更しました。

```
Type '(query: string) => output<Output> | null | undefined' is not assignable to type '(value: string) => output<Output> | null'.
  Type 'output<Output> | null | undefined' is not assignable to type 'output<Output> | null'.
    Type 'undefined' is not assignable to type 'output<Output> | null'.ts(2322)
```

```ts
import { createParser } from "nuqs";
import { z } from "zod";

function createZodCodecParser<
  Input extends z.ZodCoercedString<string> | z.ZodPipe<any, any>,
  Output extends z.ZodType
>(
  codec: z.ZodCodec<Input, Output> | z.ZodPipe<Input, Output>,
  eq: (a: z.output<Output>, b: z.output<Output>) => boolean = (a, b) => a === b
) {
  return createParser<z.output<Output>>({
    parse(query) {
      // return codec.parse(query)

      // safeParse で失敗しても null を返す
      const result = codec.safeParse(query);
      return result.success ? result.data : null;
    },
    serialize(value) {
      return codec.encode(value);
    },
    eq,
  });
}
```

次に zod codecs を定義します。今回は、`nuqs`ドキュメントの「Custom parsers」にある数値の範囲を表現する例を参考に、数値の範囲を扱えるようなスキーマを定義します。

https://nuqs.dev/docs/parsers/making-your-own#custom-multi-parsers

URLSearchParams の値として`1~2`というような文字列を渡し、state としては`{ min: 1, max: 2 }`のようにオブジェクトとして扱えるようにします。

```ts
const numberRangeCodec = z.codec(
  z.string(),
  z.object({
    min: z.number().nullable(),
    max: z.number().nullable(),
  }),
  {
    // decode は URLSearchParams から値を受け取ってオブジェクトなどにパースする関数と考える
    // Custom parsers の例ではparse()に当たる部分
    // なので、引数はstring型
    decode: (query) => {
      const [min = null, max = null] = query.split("~").map(Number);
      return { min, max };
    },
    // encode は値を受け取ってURLSearchParams に文字列を設定する、と考える
    // Custom parsers serialize()に当たる部分
    // なので、返り値がstring型
    encode: (value) => {
      return `${value.min ?? ""}~${value.max ?? ""}`;
    },
  }
);
```

### コンポーネントで使ってみる

`useQueryState`は`nuqs`のフックで、クエリパラメータの値を state として扱うことができます。

第一引数がキー名になり、第二引数にパーサーを渡すことができます。[`nuqs`には組み込みのパーサー](https://nuqs.dev/docs/parsers/built-in)がいくつも用意されているので、こちらで十分な場合も多々あると思います。

下限値と上限値を入力する`input`と、parse されたオブジェクトを表示する`pre`を設置しました。

```ts
const numberRangeParser = createZodCodecParser(numberRangeCodec);

function Demo() {
  const [numberRange, setNumberRange] = useQueryState(
    "numberRange",
    numberRangeParser
  );

  return (
    <div className="py-6 px-20 w-100">
      <div className="flex gap-4">
        <div>
          <label htmlFor="min">min</label>
          <input
            type="number"
            value={numberRange?.min ?? ""}
            max={numberRange?.max ?? 100}
            onChange={(e) =>
              setNumberRange((prev) => ({
                min: Number(e.target.value),
                max: prev?.max ?? null,
              }))
            }
          />
        </div>

        <div>
          <label htmlFor="max">max</label>
          <input
            type="number"
            value={numberRange?.max ?? ""}
            min={numberRange?.min ?? 0}
            onChange={(e) =>
              setNumberRange((prev) => ({
                min: prev?.min ?? null,
                max: Number(e.target.value),
              }))
            }
          />
        </div>
      </div>

      <pre className="bg-gray-900 text-white mt-6 rounded-sm p-4 font-normal">
        <code>{JSON.stringify(numberRange)}</code>
      </pre>
    </div>
  );
}
```

## 完成

実際に動かしてみたのが下記の GIF です。（スローになって長尺ですみません。）

![](https://storage.googleapis.com/zenn-user-upload/59dfd31cd231-20251216.gif)

### 最終的なコード

```tsx
import { useQueryState, createParser } from "nuqs";
import { z } from "zod";

function createZodCodecParser<
  Input extends z.ZodCoercedString<string> | z.ZodPipe<any, any>,
  Output extends z.ZodType
>(
  codec: z.ZodCodec<Input, Output> | z.ZodPipe<Input, Output>,
  eq: (a: z.output<Output>, b: z.output<Output>) => boolean = (a, b) => a === b
) {
  return createParser<z.output<Output>>({
    parse(query) {
      // return codec.parse(query)

      // safeParse で失敗しても null を返す
      const result = codec.safeParse(query);
      return result.success ? result.data : null;
    },
    serialize(value) {
      return codec.encode(value);
    },
    eq,
  });
}

const numberRangeCodec = z.codec(
  z.string(),
  z.object({
    min: z.number().nullable(),
    max: z.number().nullable(),
  }),
  {
    decode: (query) => {
      const [min = null, max = null] = query.split("~").map(Number);
      return { min, max };
    },
    encode: (value) => {
      return `${value.min ?? ""}~${value.max ?? ""}`;
    },
  }
);

const numberRangeParser = createZodCodecParser(numberRangeCodec);

function Demo() {
  const [numberRange, setNumberRange] = useQueryState(
    "numberRange",
    numberRangeParser
  );

  return (
    <div className="py-6 px-20 w-100">
      <div className="flex gap-4">
        <div>
          <label htmlFor="min">min</label>
          <input
            type="number"
            value={numberRange?.min ?? ""}
            max={numberRange?.max ?? 100}
            onChange={(e) =>
              setNumberRange((prev) => ({
                min: Number(e.target.value),
                max: prev?.max ?? null,
              }))
            }
          />
        </div>

        <div>
          <label htmlFor="max">max</label>
          <input
            type="number"
            value={numberRange?.max ?? ""}
            min={numberRange?.min ?? 0}
            onChange={(e) =>
              setNumberRange((prev) => ({
                min: prev?.min ?? null,
                max: Number(e.target.value),
              }))
            }
          />
        </div>
      </div>

      <pre className="bg-gray-900 text-white mt-6 rounded-sm p-4 font-normal">
        <code>{JSON.stringify(numberRange)}</code>
      </pre>
    </div>
  );
}
```

## まとめ

実際にコードを書くと`createZodCodecParser`は、いくつかの単純なパターンでの`z.codec`と組み合わせて再利用が可能なことがわかりました。

今回はネストが浅く、オブジェクト <-> 文字列 の変換を扱いましたが、実務のアプリケーションでは、より深いネストを扱う必要があったり、もっと巨大な JSON をパラメータとして扱う必要があったりするかもしれません。

そんな場合、`nuqs`の Custom parsers と Zod Codecs を組み合わせが、コードの可読性と保守性を高める助けになると少し希望を感じました。何より、URLSearchParams の文字列変換の処理が`z.codec`で宣言できるので、慣れてしまえば、どこでどのような変換が行われているかを把握しやすくなると思います。

今まで書いておいて何ですが、本当は組み込みのパーサーで済ませたいです 笑
