+++
title = 'GraalVM のバイナリ生成プロセスをデバッグ実行する'
date = 2024-10-24T17:13:55+09:00
slug = 'd671ee72f6e63699662a7d0467cc9f44'
tags = ['GraalVM']
showtoc = true
+++
GraalVM によるバイナリ生成がどのように行われているのかを調べるうえで動かしながら確認するということがしたくなってくる。そこで今回はバイナリ生成プロセスのデバッグ実行方法についてまとめる。

## 環境

- OS: Linux (WSL 2 + Ubuntu 24.04.1)
- GraalVM: 24.2.0-dev (<a href="https://github.com/oracle/graal/tree/c6e76a5740dc77b4897775ab666cbf394348d483" target="_blank">c6e76a5740dc</a>)
- JDK: 21.0.2 (<a href="https://github.com/graalvm/labs-openjdk-21/releases/tag/jvmci-23.1-b33" target="_blank">jvmci-23.1-b33</a>)
- mx: 7.33.0
- IntelliJ IDEA: 2024.2.1 (Community Edition)

## mx のデバッグオプション

まずは mx を用いたデバッグ方法について。mx コマンドのヘルプを出力すると以下のデバッグ用のオプションが見つかる。

```
$ mx
Welcome to Mx version 7.33.0
...
options:
  ...
  --dbg <address>               make Java processes wait on [<host>:]<port> for a debugger
  -d                            alias for "-dbg 8000"
  ...
```

このオプションをつけて実行すると以下の通り 8000 番ポートでの接続待ち状態で停止する。

```
$ mx -d native-image HelloWorld
Listening for transport dt_socket at address: 8000
```

あとはデバッガをアタッチすればデバッグ実行が可能となるのだが、その前に IDE 側の準備も済ませておく必要がある。今回のデバッグの話に限らず、mx には GraalVM プロジェクトを IDE で扱うためのコマンドが用意されている。

```
$ mx help | grep '(re)generate'
 eclipseinit          (re)generate Eclipse project configurations and working sets
 ideinit              (re)generate IDE project configurations
 intellijinit         (re)generate Intellij project configurations
 netbeansinit         (re)generate NetBeans project configurations
 vscodeinit           (re)generate Eclipse project configurations and working sets for VSCode usage
```

筆者は IntelliJ IDEA を使用しているため、`intellijinit` コマンドを使用する。なお、今回は substratevm プロジェクトを IDE で開きたいため、substratevm ディレクトリ配下でコマンドを実行している。

```
$ mx intellijinit
```

その後 IntelliJ IDEA を起動し、substratevm ディレクトリを選択して開くと以下の通り substratevm プロジェクトを開くことができる。依存関係も解決されているためコードジャンプなども問題なく使うことができる。

![20241024-debugging-binary-generation-01.png](../image/20241024-debugging-binary-generation-01.png)

この辺の IDE 周りの話についてはドキュメントが用意されているのでより詳しく知りたい人はこちらを参照するとよい。ただ、コードを読むだけであれば上記のコマンドを実行しておくだけで十分ではある。

- <a href="https://github.com/graalvm/mx/blob/master/docs/IDE.md" target="_blank">Loading the Project into IDEs</a>

デバッグの話に戻ると、IntelliJ IDEA の右上に GraalDebug という文字が表示されているのがわかると思うが、こちらが実はアタッチ用のデバッグ構成となる。先ほどの `intellijinit` コマンドにより作成されており、中身は以下の通り。localhost の 8000 番ポートに接続する設定となっている。

![20241024-debugging-binary-generation-02.png](../image/20241024-debugging-binary-generation-02.png)

つまりこちらを実行することで先ほど接続待ち状態となっていた `native-image` コマンドの実行プロセスにアタッチすることができ、IDE 上でデバッグ実行が可能となる。試しに `NativeImage` クラスの `main` メソッドにブレークポイントを設定して実行した結果が以下。ちゃんと停止している。ステップ実行などももちろん可能。

![20241024-debugging-binary-generation-03.png](../image/20241024-debugging-binary-generation-03.png)

これで無事デバッグ実行ができるようになったわけだが、実はこの方法では途中までしかデバッグ実行をすることができない。なぜなら、<a href="https://gingk1212.github.io/posts/a29e6d8462910862f392aa7e0bc07af9/" target="_blank">native-image ファイルの正体</a> の記事でも触れたように `NativeImage` クラスの処理の中でさらに別のプロセスを起動し java コマンドを実行しているからである。せっかくアタッチしたデバッガも別プロセスとなると追跡することができなくなる。実際に以下の java コマンドを実行している行をステップ実行するとバイナリ生成がそのまま最後まで進んでしまう。

![20241024-debugging-binary-generation-04.png](../image/20241024-debugging-binary-generation-04.png)

これでは最も関心のあるバイナリ生成プロセスの本丸の部分をデバッグ実行することができない。このままでは困ってしまうがこれに関しては心配しなくても大丈夫で、ちゃんと方法が用意されている。

## native-image のデバッグオプション

今度は native-image コマンドのヘルプを眺めてみると、こちらはこちらでデバッグ用のオプションが見つかる。`--help` では出力されないので注意。なお、このオプションについてはドキュメント内の <a href="https://github.com/oracle/graal/blob/c6e76a5740dc77b4897775ab666cbf394348d483/docs/reference-manual/native-image/contribute/DevelopingNativeImage.md#build-native-executables" target="_blank">Build Native Executables</a> という項目にも記載がある。

```
$ native-image --help-extra
Non-standard options help:
    ...
    --debug-attach[=<port or host:port (* can be used as host meaning bind to all interfaces)>]
                          attach to debugger during image building (default port is 8000)
    ...
```

オプションをつけて実行すると、先ほどと同様に接続待ち状態で停止する。

```
$ native-image --debug-attach HelloWorld
Listening for transport dt_socket at address: 8000
```

この後についても先ほどと同様で、IDE で GraalDebug を実行してアタッチするという流れ。ただしこちらは **`NativeImage` クラスにより起動されたプロセス**に対してアタッチするため、mx 側のデバッグオプションを使用した場合は追えなかった部分について追跡することができる。試しに `NativeImage` クラス内で java コマンドにより起動している `NativeImageGeneratorRunner` クラス の `main` メソッドにブレークポイントを設定して実行した結果が以下。ちゃんと停止している。

![20241024-debugging-binary-generation-05.png](../image/20241024-debugging-binary-generation-05.png)


また、native-image コマンドには `--parallelism` というスレッド数を指定するオプションがある。デバッグ実行する際にスレッド数を 1 にして追いやすくしたい、というときにこのオプションが使えるのでメモとして残しておく。

```
$ native-image --help
...
where options include:
    ...
    --parallelism         the maximum number of threads to use concurrently during native
    ...
```

## おわりに

今回はバイナリ生成プロセスのデバッグ実行方法を紹介した。デバッグ用のオプションや IDE 側のデバッグ構成が用意されているので非常にとっかかりやすくなっている。これで GraalVM の挙動をずいぶんと追いやすくなったので中身についてもっと深掘りしていきたい。
