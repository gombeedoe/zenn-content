---
title: "Motion(旧:Framer Motion)でアニメーションが動かない時は「視差効果を減らす」設定を確認する"
emoji: "🐎"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["motion", "framermotion", "javascript", "css", "react"]
publication_name: "galapagos"
published: false
---

この記事は[株式会社ガラパゴス（有志）アドベントカレンダー 2025](https://qiita.com/advent-calendar/2025/galapagos)の 14 日目の記事です。

## 実装したアニメーションが動かない時がある

ビューポート（表示領域）外にある要素に対して、ページ読み込みと同時にスライドインするアニメーションでビューポート内に表示したい、という場面がありました。

React などのコンポーネントライブラリは使っておらず、純粋な HTML、CSS、JavaScript を使って実装していました。

React を使ったプロジェクトで使っていたこともあり、Motion（旧：Framer Motion）の JavaScript バージョンを使って先述のアニメーションを実装しました。

自分の環境で一通り確認して、「よし OK」と思って共有をしたら、「アニメーションが動かない」と連絡がありました。しかも、複数人の環境でアニメーション実行の不発が確認されました。特に困っていたのは、アニメーションの実装は同じコードを使っているのに、スマホ環境だけで動かないという点でした。

自分の環境で再現できず結構困っていたところで、下記の記事に出会いました。

[アニメーションが効かない原因は「視差効果を減らす」かも？そしてアクセシビリティへ](https://zenn.dev/tam_tam/articles/4c98d8f8098797)

## 「視差効果を減らす」の設定を確認する

先の記事では、リセット用の CSS で`@media (prefers-reduced-motion: reduce) {}`というメディアクエリでアニメーションが制限される記述があり、その影響でアニメーションが動かないとのことでした。

早速自分の環境でも **「視差効果を減らす」** をオンにしてみました。
Mac だと、「システム設定」>「アクセシビリティ」> 「ディスプレイ」のリストの中にあります。（画像参照、Mac OS のバージョンで変わるかもしれません）

![](https://storage.googleapis.com/zenn-user-upload/c1dd3b9ed2a4-20251212.png)

見事に自分の環境でもアニメーションが動かない事象を再現できました。

しかし、下記の謎が残っていました。

- 自分が使ったリセット用の CSS では、`@media (prefers-reduced-motion: reduce) {}`でのアニメーション制限は存在しない。
- スライドインのアニメーションは動かないが、透明度の変化（フェードイン）は動いている。

## Motion でのアニメーション制限を探る

CSS に原因がなさそうだったので、Motion の方を探ってみることにしました。

今回、JavaScript バージョンの Motion を使っていたので盲点でしたが、React バージョンのドキュメントの方に下記のページがありました。

[Create accessible animations in React — Guide | Motion](https://motion.dev/docs/react-accessibility)

> アニメーションはユーザビリティに重大な影響を及ぼす可能性があり、人によっては乗り物酔いを引き起こすこともあります。
> （中略）
> Motion for React は、こうしたユーザーの好みを尊重するための API を提供しています。このガイドでは、`reducedMotion`オプションと`useReducedMotion`フックを使ってアニメーションをアクセシブルにする方法を学びます。
> （中略）
>
> `reducedMotion` を `"user"` に設定すると、すべてのモーション コンポーネントで変換とレイアウト アニメーションが自動的に無効になりますが、不透明度や backgroundColor などの他の値のアニメーションは保持されます。

```jsx
import { MotionConfig } from "framer-motion";

export function App({ children }) {
  return <MotionConfig reducedMotion="user">{children}</MotionConfig>;
}
```

なるほど、Motion for React には、`reducedMotion`というオプションで「視差効果を減らす」の設定を反映させることができるようです。

しかし、[reducedMotion](https://motion.dev/docs/react-motion-config#reducedmotion)の項目を確認してみると、

> Default: `"never"`

になっており、デフォルトではアニメーションの制限は行われないようです。

では、なぜ自分の実装ではアニメーションが制限されているのでしょうか？JavaScript バージョンと React バージョンの違いがあるのだろうか？

ということで、「視差効果を減らす」をオンにした状態で、Motion の Examples ページを確認してみました。それぞれ「Rotate」（回転）の例を確認してみます。

どちらの例も **「Reduced Motion is enabled. Examples might not work as expected.」** と表示されるようになりました。オフの時は出ない表示です。わかりにくいですが、該当画面をキャプチャした GIF 画像を添付します。

まずはこちらが React の例です。

https://motion.dev/examples/react-rotate

![](https://storage.googleapis.com/zenn-user-upload/bdd578fd1c53-20251212.gif)

次にこちらが JavaScript の例です。

https://motion.dev/examples/js-rotate?platform=js

![](https://storage.googleapis.com/zenn-user-upload/e52c62b49dac-20251212.gif)

### JavaScript バージョンは動いていない

お分かりいただけただろうか。（一度言ってみたかったフレーズです。）

React バージョンは動いていたのに、JavaScript バージョンは動いていません。（JS の方で GIF がチラつくのはブラウザのリロードです。）

ということで、React バージョンと JavaScript バージョンではデフォルト値が異なることがわかりました。正確なドキュメントを発見できなかったのですが、少なくとも JS 版の Examples では 「視差効果を減らす」有効時に transform 系が抑制される挙動を確認できました。

では、「JavaScript バージョンの設定を変えよう」と思ったのですが、React バージョンの`<MotionConfig />`にあたる config を設定する方法が見当たりません。

これは困りました。

## 最終的な対応は「視差効果を減らす」がオン時のアニメーションを用意した。

> `reducedMotion` を `"user"` に設定すると、すべてのモーション コンポーネントで変換とレイアウト アニメーションが自動的に無効になりますが、不透明度や backgroundColor などの他の値のアニメーションは保持されます。

JavaScript バージョンでも透明度（フェードイン）のアニメーションは維持されるようで、どうやら上記の React バージョンの `"user"` に設定した時の同じ挙動と予測できます。

なので、最終的には「視差効果を減らす」がオン時は代替のアニメーションを用意することにしました。

```js
import { animate } from "https://cdn.jsdelivr.net/npm/motion@12.23.24/+esm";

// ユーザーがアニメーションを減らす設定をしているか確認
const prefersReducedMotion = window.matchMedia(
  "(prefers-reduced-motion: reduce)"
).matches;

// 減らす場合は、[0, 1]を渡して透明度をアニメーションさせる
const opacityTransition = prefersReducedMotion ? [0, 1] : 1;

animate(
  ".js-animated-element",
  { right: ["100%", "-100%"], opacity: opacityTransition },
  { duration: 5, ease: "easeInOut" }
);
```

上記のように、`window.matchMedia("(prefers-reduced-motion: reduce)").matches`でユーザーが「視差効果を減らす」がオンになっているかを判定して、`prefersReducedMotion ? [0, 1] : 1`で、オンの時は透明度のアニメーションを行うようにしました。

こうすることで、オフの時はスライドイン、オンの時はフェードインのアニメーションを適用できます。

## なぜ`prefers-reduced-motion`が必要かを説明する必要がある

MDN にも下記の説明があります。

[prefers-reduced-motion - CSS | MDN](https://developer.mozilla.org/ja/docs/Web/CSS/Reference/At-rules/@media/prefers-reduced-motion)

> prefers-reduced-motion は CSS のメディア特性で、ユーザーが余計な動きを最少化するよう要求したことを検出するために使用します。この設定は、ユーザーがモーションベースのアニメーションを削除、縮小、または置き換えるインターフェイスを推奨していることを、端末のブラウザーに伝えるために使用します。
>
> このようなアニメーションは、前庭運動障碍のある人に不快感を引き起こす可能性があります。大きなオブジェクトを拡大縮小したりパンなどしたりするアニメーションは、前庭運動を引き起こす可能性があります。

ユーザーが「視差効果を減らす」をオンにしているということは、アニメーションを減らす、または置き換えることを望んでいるということです。ですので、ユーザーの`prefers-reduced-motion`や Motion の config を利用してユーザーの設定を尊重するのはとても大事だと思います。そういった意味で、JavaScript バージョンの Motion のデフォルト設定は正しいと思います。

あとは、ベンダーやクライアントに事前にこのことを説明すること（できること）が大事なのだと今回痛感しました。
