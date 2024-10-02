+++
title = 'GitHub Pages と Hugo でブログを作った'
date = 2024-09-25T21:50:45+09:00
slug = '4b9e2a244127ee1ea18b49cb3a1e6807'
tags = ['blog']
showtoc = true
+++
もともとはてなブログで <a href="https://penguing27.hatenablog.jp/" target="_blank">gingk’s blog</a> というブログを持っていたが、特にスマホで見たときに広告が多すぎて見づらいと思っていたので心機一転新しく開設することにした。とはいえ見ればわかる通りでこれまでたくさん書いてきたのかと言われればまったくもって No であり、そのあたりもこっちに移行することでもっと気軽に書けるようになるのではないかという狙いもある。

このポストではこのブログを公開するまでの手順を書く。リポジトリはこちら。  
<a href="https://github.com/gingk1212/gingk1212.github.io" target="_blank">gingk1212/gingk1212.github.io</a>

## Hugo 導入

日々いろんな人のブログやサイトを見ながらなんとなく GitHub Pages で作りたいという思いがあったので、GitHub Pages でブログを作るためにはどうすればいいかをまずは調べた。で、選択肢として多く目に入ったものが Jekyll と Hugo という静的サイトジェネレータ。Jekyll は Ruby で、Hugo は Go で書かれているみたいだが、比較したわけではないけど Hugo のほうがなんとなく速そうだし今風に思えたので Hugo をまずは触ってみることにした。

こちらに従いインストール。Windows を使っているのだけど、パッケージマネージャーは流行り廃りがあってあまり使いたい気分ではなかったのでビルド済みのバイナリをダウンロードしてパスを通しておいた。  
<a href="https://gohugo.io/categories/installation/" target="_blank">Installation | Hugo</a>

Quick start に従って動かしてみた。  
<a href="https://gohugo.io/getting-started/quick-start/" target="_blank">Quick start | Hugo</a>

## テーマ選び

特に違和感もなかったのでこのまま Hugo を使うことにしてテーマ選びを開始。  
<a href="https://themes.gohugo.io/" target="_blank">Complete List | Hugo Themes</a>

ここに一番時間がかかった。ただただシンプルな見た目でいいと思っていたのだけど、意外とこれってものが見つからずに延々といろんなテーマを試しまくることになった。結果採用したのが <a href="https://github.com/adityatelange/hugo-PaperMod" target="_blank">PaperMod</a>。見た目もシンプルで気に入っているが、何よりよかったのがドキュメントが丁寧なところ。見た目はいい感じだけど設定をどう書けばいいかいまいちわからず泣く泣く断念した、というものもけっこうあった。PaperMod は Wiki に丁寧に書いてくれているので、まずは <a href="https://github.com/adityatelange/hugo-PaperMod/wiki/Installation#sample-configyml" target="_blank">サンプルの config</a> をコピペして持ってきて、Wiki や <a href="https://adityatelange.github.io/hugo-PaperMod/" target="_blank">デモサイト</a> (<a href="https://github.com/adityatelange/hugo-PaperMod/tree/exampleSite" target="_blank">リポジトリ</a>) を見ながら手を加えていった。

## 公開

公開については Hugo がドキュメントを用意してくれているのでその通りにやった。  
<a href="https://gohugo.io/hosting-and-deployment/hosting-on-github/" target="_blank">Host on GitHub Pages | Hugo</a>

GitHub Actions を使うのが地味に初めてだったのでプッシュした後動いているのを見て軽く感動した。

## Google Analytics, Google Search Console 導入

はてなブログでもやっていたのだけど、どうせなら何人アクセスしているか知りたいし検索にも引っかかってほしいので今回も導入した。ググればやり方はいくらでも出てくるので詳細は割愛。

なお、Hugo で Google Analytics を有効にするための設定方法はこちら。  
<a href="https://gohugo.io/templates/embedded/#configure-google-analytics" target="_blank">Configure Google Analytics</a>

Google Analytics 導入に伴い、プライバシーポリシーもきちんと配置した。リンクをどこに置くか迷ったのだけど、右上に置くのはちょっと目立ちすぎかと思いフッターに配置することにした。本来の使い方とは違ってしまうが、`Copyright` 項目に文字を書くことでフッターに表示されるのでお借りすることにした。  
<a href="https://gohugo.io/methods/site/copyright/" target="_blank">Copyright | Hugo</a>

## URL について

最後に、記事の URL について。デフォルトだと以下だが、ファイル名なんてものはフォルダ構成再検討などに伴って将来絶対に変えたくなってくるものでありその都度 URL が変わってしまうのは嫌だったので別の案を検討。  
`https://gingk1212.github.io/posts/ファイル名(拡張子除く)/`

記事のファイル内で <a href="https://gohugo.io/content-management/urls/#slug" target="_blank">`slug`</a> を設定することで URL のパス名にすることができるようなので、`slug` にランダム文字列を設定することにした。ランダム文字列はいろいろググっていると <a href="https://gohugo.io/methods/page/file/#uniqueid" target="_blank">`UniqueID`</a> が使えるということがわかったのでこちらを使用。ランダム文字列というかファイルパスのハッシュ値だが、今回の目的においては十分である。

ちなみに `posts/` 配下に置くファイルの名前やフォルダ構成だが、年月日でフォルダを切って配置、などなどいろいろ考えたが運用してみないことには見えてこないのでとりあえずはフラットに置いていくことにした。ただ最低限日付順に並んでほしくはあるのでファイル名に日付を入れるようにはしている。具体的には、下記のコマンドを使用してファイルを生成している。

```
$ hugo new content content/posts/`date '+%Y%m%d'`-create-blog.md
```

## 終わりに

半分思い付きで作ってみたわけだけど、デザインやレスポンスの速さなどもろもろ大変満足している。GitHub Pages x Hugo は世の中に知見が大量にたまっているので特に詰まることなくすんなり作ることができた。こういうのは作った時が一番モチベーションが高くその後どんどん下がっていくものなので、へこたれずに続けていきたい。



