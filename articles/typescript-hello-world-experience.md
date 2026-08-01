---
title: "CDKでつまずいたので、TypeScriptの基礎からHello Worldをやり直した話"
emoji: "🔰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["typescript", "nodejs", "初学者"]
published: true
---

## はじめに

昨日AWS CDKを触っていたが、TypeScriptの文法自体をよく理解しないまま書いていたため「よくわからないまま終わった」状態になってしまった。CDKはTypeScriptの上に成り立つツールなので、土台となる言語仕様を飛ばしたままでは応用が効かないと感じ、今日は一度立ち止まってTypeScriptの基礎、いわゆるHello Worldからやり直すことにした。

この記事では、環境構築からコンパイル・実行までの一連の流れと、その過程で理解した`console.log`・`tsc`の役割、`let`と`const`の違いについてまとめる。

## 環境構築

まずはプロジェクトを作り、TypeScriptをインストールする。

```bash
npm init -y
npm install -D typescript
```

`-D`（`--save-dev`）を付けているのは、TypeScriptが実行時に必要なランタイム依存ではなく、開発・ビルド時にのみ使うツールだからである。

続けて`tsconfig.json`を生成する。

```bash
npx tsc --init
```

`tsconfig.json`はTypeScriptコンパイラ（`tsc`）の挙動を設定するファイルで、どのバージョンのJavaScriptに変換するか（`target`）、出力先ディレクトリ（`outDir`）などをここで指定する。

## Hello Worldを書く

`src/index.ts`を作成し、以下を書いた。

```typescript
console.log("Hello, World!");
```

`console.log`は「画面（標準出力）に書き出す」命令であり、カッコの中身は実行結果を確認するためのメモのようなもので、文字列でも変数でも自由に渡せる。今回は最もシンプルな文字列リテラルを渡している。

## コンパイルして実行する

TypeScriptのファイルはそのままでは実行できない。`tsc`でJavaScriptに変換（コンパイル）してから、Node.jsで実行する。

```bash
npx tsc
node dist/index.js
```

```
Hello, World!
```

ここで整理できたのは、TypeScriptとJavaScriptの役割の違いである。

- TypeScriptは型注釈などを含む、**人間が書きやすく・読みやすいための言語**
- JavaScriptは**機械（ブラウザやNode.jsのエンジン）が実行するための言語**
- `tsc`はその間を取り持つ「翻訳者」であり、型情報を取り除きつつ構文をJavaScriptに変換する

CDKで見ていたコードも、最終的にはこの`tsc`によってJavaScriptに変換されてから実行される、という構造を理解できたのが今日一番の収穫だった。

## letとconstの違いを、エラーを起こして体験する

`let`と`const`の違いは「再代入できるかどうか」だとドキュメントには書いてあるが、実際に試してみることにした。

```typescript
const message = "Hello, World!";
message = "Hello, TypeScript!"; // 再代入してみる
```

これをコンパイルすると、以下のエラーが発生した。

```
error TS2588: Cannot assign to 'message' because it is a constant.
```

`const`で宣言した変数へ再代入しようとすると、コンパイルの時点（実行前）で弾かれることを確認できた。実行時エラーではなくコンパイルエラーとして検出される点も、TypeScriptらしい挙動だと感じた。

一方で`let`で宣言し直すと、同じ再代入は問題なく通る。

```typescript
let message = "Hello, World!";
message = "Hello, TypeScript!"; // エラーにならない
```

この体験から、実務でよく言われる「**基本は`const`を使い、値を変更する必要がある場合のみ`let`を使う**」という定石の意味を、単なるルールとしてではなく腹落ちした形で理解できた。`const`をデフォルトにしておくことで、「この変数は途中で変わらない」という前提をコード自体が保証してくれる、という安心感がある。

## まとめ

- `npm init` → TypeScriptインストール → `tsconfig.json`設定 → `.ts`を書く → `tsc`でコンパイル → `node`で実行、という一連の流れを体験した
- `console.log`は標準出力への書き出し命令であり、TypeScriptは人間向け・JavaScriptは機械向けの言語で、両者を`tsc`が翻訳している
- `const`への再代入は`TS2588`としてコンパイル時に検出される
- 「基本は`const`、必要な時だけ`let`」という定石を、実際にエラーを起こすことで体感できた

CDKでつまずいた原因の一端は、この程度の基礎を素通りしていたことにあったと感じている。次はCDKのコードに戻り、今日理解した内容がどう活きているかを確認していきたい。
