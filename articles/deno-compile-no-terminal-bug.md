---
title: "`deno compile --no-terminal`を使いたいときはmacOSを使おう"
emoji: "🍎"
type: "tech"
topics: [deno]
published: true
---
この記事は、Deno Advent Calendar 2023の18日目です。
https://qiita.com/advent-calendar/2023/deno

## `deno compile`とは
Deno本体が有する、JavaScirptファイルを単一実行可能ファイルにコンパイルするコマンドです。

https://docs.deno.com/runtime/manual/tools/compiler

`--target`を付与することでWindowsやmacOS、Linux向けにコンパイルできるため、コンパイルするのにわざわざそのOSを用意する必要もありません。  
`--include`を付与することでWorkersを含めてコンパイルすることもできます。  
詳細は上記の公式ドキュメントを御覧ください。

ちなみに、Node.jsも[v19.7.0](https://github.com/nodejs/node/releases/tag/v19.7.0)で単一実行可能ファイルのコンパイルに対応しました。  
これはNode.jsのリリースもやられているRuy Adornoさんのスライドにも書かれています。

https://speakerdeck.com/ruyadorno/the-node-dot-js-runtime-renaissance?slide=56

[担当チームのブログ](https://github.com/nodejs/single-executable/blob/main/blog/2022-08-05-an-overview-of-the-current-state.md)では、Denoが引き合いに出されていました。

## `deno compile --no-terminal`とは
デフォルトでは、Windowsに向けて`deno compile`をするとターミナルが背後に表示されるようになっています。

![Untitled](/images/deno-compile-no-terminal-bug/show-terminal.png)

これはWindowsの単一実行ファイルのヘッダーにある[Subsystemフィールド](https://learn.microsoft.com/ja-jp/windows/win32/debug/pe-format#windows-subsystem)の部分が2になっていなかったのが原因のようで、以下のPRで修正され、[v1.36.0](https://github.com/denoland/deno/releases/tag/v1.36.0)で`--no-terminal`が追加されました。

https://github.com/denoland/deno/pull/17991

しかし、v1.39.0現在、このフラグは正常に動作していません。

## WindowsとWSLで`--no-terminal`をつけるとアプリが表示されない
検証当時はv1.38.0で、Ubuntu on WSL 2でコンパイルしたところ発覚したためIssueを提出しました。

https://github.com/denoland/deno/issues/21091

その後、v1.39.0とリリース時のv1.36.0でコンパイル結果を確認しましたが、私の環境下ではWindows 10とUbuntu on WSL 2でアプリが表示されないという事象が確認されました。
macOSであれば問題ないようです。

https://github.com/denoland/deno/issues/21091#issuecomment-1859182338 

## GitHub Actionsで解決
せっかく`deno compile`が手持ちのOSに依存しない仕組みになっていても、ここで『macOSでしか`--no-terminal`が動かないから実機を準備』となっては意味はありません。  
そこで、手元にないmacOS環境に頼ります。

https://docs.github.com/ja/actions/using-github-hosted-runners/about-github-hosted-runners/about-github-hosted-runners#supported-runners-and-hardware-resources

GitHub ActionsはmacOSをサポートしているため、わざわざ実機を準備する必要はありません。  
この環境を利用することで、`--no-terminal`が適用されたアプリが表示されるはずです。

## 最後に
以上が、私が遭遇した`deno compile`の挙動の話でした。  
近いうちに、この挙動に遭遇したときの制作物についても書きたいと思います。

誰かの開発の一助となれば幸いです。  
お読みいただき、ありがとうございました。
