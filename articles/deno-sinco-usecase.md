---
title: "\/// <reference ...>を用いてSincoを使ったE2E試験での型エラーを解決する"
emoji: "🦕"
type: "tech"
topics: [deno, sinco]
published: true
---

この記事は、Deno Advent Calendar 2021の21日目です。

https://qiita.com/advent-calendar/2021/deno/

Zennでは初めての記事になります。よろしくお願いします。

## はじめに

DenoでE2E試験をするには、[deno-puppeteer](https://deno.land/x/puppeteer@9.0.2)と[Sinco](https://deno.land/x/sinco@v3.1.0)の大きく分けて2つがあると思います。
Chrome等を事前インストールをしない点で私はSincoを気に入っており、今回利用しました。
その中で、試験で発生した型エラーと、その回避策を共有します。

## この記事で伝えたいこと

- TypeScriptでSincoを利用するときに、プロジェクトルートにtsconfig.jsonを使うことで起きる型エラーの解決と実装への影響
- トリプルスラッシュディレクティブ（`/// <reference ...>`）を利用することで実装への影響を回避する方法

## Sincoで要素名などを取るときに起こる問題

Sincoで要素名を取るときは、`Page.evaluate()`を利用して、以下のように書きます。

```typescript
const sinco = await buildFor("chrome");
const page = await sinco.goTo("https://example.com/");
const pageTitle: string = await page.evaluate(() => {
  return document.querySelector("h1")?.innerText;
});
```

この`pageTitle`の中に今回のURLでいうと`Example Domain`が入るはずですが、このままですとdocumentの下に赤い線が引かれ、documentの型が見当たらないという警告が出ます。

![](/images/deno-sinco-usecase/sinco-type-err.png)

これは、Denoの型定義の中にdocumentが存在しないためです。

そこで、[`Page.evaluate()`](https://drash.land/sinco/v3.x/tutorials/page/evaluate)の公式ドキュメントを参考に、プロジェクトルートに`tsconfig.json`を作成します。

```json
{
  "compilerOptions": {
    "lib": ["dom", "dom.asynciterable", "deno.ns"]
  }
}
```

[`deno.ns`](https://github.com/denoland/deno.ns)はDenoが提供するDenoネームスペースのNode.jsモジュールです。これを`lib`に指定することで、`Deno.test()`といったメソッドが利用可能になります。
なお、`dom.asynciterable`は私の場合では必要だったのですが、人によってはいらないかもしれません。

![](/images/deno-sinco-usecase/sinco-allgreen.png)

これで、問題なく試験ができる環境になりました。

![](/images/deno-sinco-usecase/sinco-test-pass.png)

さぁ、これで解決！ ……というわけではありません。
実装を開くと、例えばURL Pattern APIが`any`になっていたり、人によっては他にも`any`になっているものが出てくるかもしれません。

## なぜ実装でanyが出るのか

これはあくまで推測になりますが、URL Pattern APIが一部のChromiumブラウザかDenoでしか実装されていない限定的なAPIであることで、tsconfig.jsonの設定では網羅できなかったのではないかと考えています。

https://developer.mozilla.org/en-US/docs/Web/API/URL_Pattern_API#browser_compatibility

DenoはWeb標準をいち早く実装する流れがあるため、こういった事例は一定数起こりうるのではないでしょうか。
これを解決するために、トリプルスラッシュディレクティブというものを利用します。

## トリプルスラッシュディレクティブとは

TypeScirptが提供している型定義の反映方法です。ファイルの先頭行に`/// <reference ...>`のように記述することで利用でき、Denoでも対応しています。

https://www.typescriptlang.org/docs/handbook/triple-slash-directives.html
https://deno.land/manual@v1.17.0/typescript/types#using-the-triple-slash-reference-directive

今回利用するトリプルスラッシュディレクティブは以下の2つです。

- `/// <reference lib="..." />`
- `/// <reference no-default-lib="true"/>`

`/// <reference lib="..." />`は`tsconfig.json`の`lib`に相当するものです。たとえば`/// <reference lib="dom" />`と入力すれば、DOMの型定義が適用されます。
`/// <reference no-default-lib="true"/>`はtsconfig.jsonの`noLib`に相当するものです。`/// <reference lib="..." />`より前に記述します。ライブラリファイルの自動インクルードを無効化するようです。

## トリプルスラッシュディレクティブでエラーを解決する

それでは、E2E試験のファイルに実際に適用していきます。
ファイルの先頭行に、以下のように記述します。

```typescript
/// <reference no-default-lib="true" />
/// <reference lib="dom" />
/// <reference lib="dom.asynciterable" />
/// <reference lib="deno.ns" />
```

E2E試験のために作成した`tsconfig.json`は合わせて削除してください。
これで、実装にも試験にも型エラーはなくなったと思います。

## 最後に

この記事を思いついたのは、12月18日のことでした。たまたまURL Pattern APIのanyを見つけたのが起因です。

https://twitter.com/windchime_yk/status/1471900944602644480

そこから記事の構想が固まるまでに時間がかかったため、急ピッチの記事作成となってしまいました。できるだけ資料を読んで書いたつもりですが、認識違いなどあればご指摘ください。

今回の記事作成にあたって、[ミノ駆動](https://twitter.com/MinoDriven)さんの以下の記事を参考にさせていただきました。ありがとうございます。

https://qiita.com/MinoDriven/items/6718b5e70e3fb321ff9b

最後になりますが、トリプルスラッシュディレクティブを利用した今回のテストファイルをGitHub Gistにあげているので、興味のある方はご覧ください。

https://gist.github.com/windchime-yk/5d242d9e6a117a8f9215baabe79b109d

以下のコマンドを実行することで、実際に走らせることもできます。

``` bash
deno test --allow-net --allow-run https://gist.githubusercontent.com/windchime-yk/5d242d9e6a117a8f9215baabe79b109d/raw/4e534d8e7c7f5f4c1225b0ad1375917869b10720/deno.e2e.ts
```

誰かの開発の一助となれば幸いです。
お読みいただき、ありがとうございました。
