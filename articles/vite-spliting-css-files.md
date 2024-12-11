---
title: "Vite + ReactでCSSを分割して読み込む"
emoji: "🪵"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["vite", "react", "tailwindcss"]
published: false
---

TODO:
vite-react-ts-bun リポジトリに動作確認用に`tailwindの@base`を別書き出しにするコードを書く。

# 動機

[Ant Design v5](https://ant.design/)と CSS Modules で開発されたプロジェクトで、
途中から[Tailwind CSS](https://tailwindcss.com/)を導入したくなったのですが、
下記の要望を実現して導入する必要がありました。

- Tailwind の Preflight は off にしたくない
  - html タグにデフォルトスタイルを当てたくない
- CSS Modules で書かれたスタイルは全く崩したくない
- antd のコンポーネントが Tailwind の影響を受けてほしくない

antd が v5 からは less から CSS-in-JS(おそらく emotion)に変更されたため、下記の課題がありました。

- Tailwind の Preflight で antd のコンポーネントのスタイルが崩れてしまう。
- Tailwind の Preflight を off(`false`)にすると、html 要素にデフォルトスタイルが当たってしまう
- antd の詳細度を上げる`<StyleProvider hashPriority="high">...</StyleProvider>`は CSS Modules のスタイルを上書きしてしまう or Tialwind のユーティリティクラスが詳細度で軽くなる -（どちらだったか曖昧ですみません。ただ、`hashPriority="high"`は要望を満たせなかった。）

antd v5 と Tailwind のコンフリクトについては、下記の issue で議論されています。
https://github.com/ant-design/ant-design/issues/38794

上記の issue コメントの中で最も自分の考えにマッチしたのが下記のコメントでした。
https://github.com/ant-design/ant-design/issues/38794#issuecomment-1364501286

> 主な問題は preflieght ではなく、`<head>`内のスタイルタグの順序です。Tailwind は通常、style loader/sass loader を使って一番下に配置され、ant design は一番上に配置されます。そのため、上書きの方が優先順位が高いのです。

結局は、CSS の読み込み順を解決すれば、要望を満たせることがわかりました。(まさにカスケーディング)
