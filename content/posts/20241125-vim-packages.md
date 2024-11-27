+++
title = 'Vim のプラグイン管理にパッケージ機能を使っている'
date = 2024-11-25T09:00:19+09:00
slug = '4bc9aed16fcef50fe2cea7cff010953b'
tags = ['Vim']
showtoc = true
+++

この記事は <a href="https://vim-jp.org/ekiden/" target="_blank">Vim 駅伝</a> の 273 本目の記事です。

Vim のプラグイン管理に標準の<a href="https://vim-jp.org/vimdoc-ja/repeat.html#packages" target="_blank">パッケージ機能</a>を使っているので、どんな感じで使っているかを書いてみる。ちなみにこれまでのプラグイン管理方法の変遷は以下の通り。

<a href="https://github.com/Shougo/neobundle.vim" target="_blank">neobundle.vim</a> → <a href="https://github.com/Shougo/dein.vim" target="_blank">dein.vim</a> → パッケージ機能

.vimrc のコミット履歴を見てみたところ、2018年にパッケージ機能の乗り換えたらしい。きっかけはあまり覚えていないが、基本的に標準のものを使うのが好きなのでパッケージ機能に関する記事か何かを見て興味本位で乗り換えたんだと思う。たぶん今はもっと便利で効率的なプラグイン管理のやり方があると思うが、こんな人もいるんだなあとゆるく読んでもらえると幸いです。

## パッケージは一つだけ

パッケージ機能というだけあってパッケージは複数使うことができるが、今のところ一つしか使っていない。複数使いたくなるケースが果たしてあるのかいまいちよくわかっていない。あと、start あるいは opt という名前のディレクトリの中にプラグインを格納する必要があるのだけど、start しか使っていない。opt に置くと起動時ではなく任意のタイミングでプラグインを読み込むことができるみたいなのだけど、現状読み込みに時間のかかるプラグインは使っていないので start で事足りている。

なので、ディレクトリ構成は以下の通り。プラグインを新たに導入するときは start 配下で `git clone` している。

```
.vim/pack/
└─ mypackage
    └─ start
        ├─ plugin A
        ├─ plugin B
        ├─ ...
```

## 更新方法

パッケージ機能には当然自動更新してくれる機能なんてものはないので、プラグインを更新したければ自分で更新する必要がある。なので、気が向いたタイミングで以下のコマンドを実行することで一括で更新している。

```
$ for d in ~/.vim/pack/mypackage/start/*; do echo $d; cd $d; git pull --prune; cd -; done
```

シェルスクリプトにしてもいいのだけど、一度上記のコマンドを実行するとヒストリーに残り、以降は `Ctrl-R` → `for d` とか打つと大体このコマンドが出てきてくれるので結局シェルスクリプトは作っていない。

## ヘルプ

pack 配下にプラグインを配置してそのプラグインのヘルプを `:help ...` で見ようとしても残念ながらヘルプは表示されない。なので、~/.vim/doc に以下のスクリプトを配置し、新しいプラグインを導入したり更新したりするたびに実行している。実行後は Vim を起動して `:helptags .` を実行することでタグファイルを作成する必要もある。

```bash
#!/bin/bash

echo [rm]
ls ./*.txt
rm ./*.txt

echo [cp]
PACK="mypackage"
for file in `ls ~/.vim/pack/${PACK}/start/*/doc/*`; do
  ls ${file}
  cp -p ${file} ~/.vim/doc/
done
echo "DONE!!"
echo "Type \"vim\" and \":helptags .\""
```

正直これに関してはまあまあめんどくさいと思っているのでもっといい方法があればぜひ知りたい。この機会にちょっとググってみたところヘルプファイルを ~/.vim/doc にコピーするのではなくシンボリックリンクを作成している人もいた。少なくともそのほうがプラグイン更新の際に気にかける必要がなくなるのでよさそう。

## プラグインごとの設定の書き方

これに関してはパッケージ機能特有の話ではなさそうな気がするけど記載しておく。.vimrc にプラグインの設定を記載する際にたとえばある環境ではそのプラグインを導入しているから設定を有効化したいが、別の環境ではプラグインを入れてないので設定を無効にしておきたいというケースがある。そんなときのためにプラグインの設定は以下のように記述している。

```vimrc
"-----------------------------------------------------------
" lightline.vim
"

" Install check
if !empty(glob(s:pack_dir . '/start/lightline.vim'))

let g:lightline = {
      \ 'colorscheme': 'wombat'
      \ }

" Install check end
endif
```

つまり、ディレクトリの有無で有効にするかどうかを判定している。なお、`s:pack_dir` は以下のように定義している。

```vimrc
if has("win64") || has("win32")
  let s:pack_dir = expand('~/vimfiles/pack/mypackage')
else
  let s:pack_dir = expand('~/.vim/pack/mypackage')
endif
```

(2024.11.27 追記)  
X にて h_east さんに上記の if 文は `if has("win32")` のみで十分ということを<a href="https://x.com/h_east/status/1861308565740536089" target="_blank">教えていただいた</a>。たしかに `feature-list` のヘルプにそのような記載があった。感謝。

```
win32			Win32 version of Vim (MS-Windows 95 and later, 32 or
			64 bits)
```

## おわりに

今はせいぜい10個弱くらいしかプラグインを使っていないので、上記の方法でまあまあ問題なく運用できている。標準のパッケージ機能を使った運用の一例として、何かしら参考になればうれしいです。
