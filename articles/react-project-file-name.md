---
title: "reactプロジェクトのファイル名にケバブケースを使う"
emoji: "🦦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["react"]
published: false
---

# きっかけ

https://profy.dev/article/react-folder-structure

React のフォルダ構造を調べる都合で、上記の記事を読んだ時に下記のセクションタイトルがありました。

> kebab-case for File and Folder Names

下記、本文の Google 翻訳。

> 以前は、コンポーネント・ファイルにはパスカル・ケース（例：MyComponent.js）、関数／フックにはキャメルケース（例：useMyHook.js）で名前を付けていた。それから MacBook に乗り換えた。
> リファクタリング中に、myComponent.js を MyComponent.js にリネームした。ローカルではうまくいったが、GitHub の CI では import 文が壊れていると文句を言われた：
> import MyComponent from "./MyComponent"；
> 何時間もデバッグが続いた。MacOS は大文字と小文字を区別しないファイルシステムなので、MyComponent.js と myComponent.js は同じであることがわかった。Git はこの変更を認識しなかったが、GitHub の CI は大文字と小文字を区別する Linux のイメージを使用していたため、問題が発生した。
> これを避けるには、ファイル名とフォルダ名に kebab-case を使う：
> MyComponent.js の代わりに my-component.js と書く。
> useMyHook.js の代わりに、use-my-hook.js と書いてください。

## なぜ camelCase/PascalCase を使うか
