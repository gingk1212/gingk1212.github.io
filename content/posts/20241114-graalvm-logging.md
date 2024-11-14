+++
title = 'GraalVM 内のログ出力について'
date = 2024-11-14T17:00:46+09:00
slug = 'f68f57df24c36bd4c05c89d811303ca5'
tags = ['GraalVM']
showtoc = true
+++
GraalVM のソースコードを読んでいるとよくこういう処理を見かける。デバッグ用のログを出しているんだろうとは思っていたのだけれど、普通に native-image コマンドを実行しても出力されず気になっていたので調べた。

```java
try (Indent indent = debug.logAndIndent("create native image")) {
    ...
```

## ログを出力する方法

結論、<a href="https://github.com/oracle/graal/blob/master/compiler/docs/Debugging.md#logging" target="_blank">ここ</a>に書いてあった。一部内容を引用する。

> For more permanent logging statements, use the `log(...)` methods in
<a href="https://github.com/oracle/graal/blob/master/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/debug/DebugContext.java" target="_blank">`DebugContext`</a>.
A (nestable) debug scope is entered via one of the `scope(...)` methods in that class. For example:

```java
DebugContext debug = ...;
InstalledCode code = null;
try (Scope s = debug.scope("CodeInstall", method)) {
    code = ...
    debug.log("installed code for %s", method);
} catch (Throwable e) {
    throw debug.handle(e);
}
```

> The `debug.log` statement will send output to the console if `CodeInstall` is matched by the
`-Djdk.graal.Log` option. The matching logic for this option is implemented in
<a href="https://github.com/oracle/graal/blob/master/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/debug/DebugFilter.java" target="_blank">DebugFilter</a>
and documented in the
<a href="https://github.com/oracle/graal/blob/master/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/debug/doc-files/DumpHelp.txt" target="_blank">Dump help message</a>.

つまり、`-Djdk.graal.Log` オプションを使ってスコープ名を指定すると、そのスコープ内で出力されているログがコンソールに表示されるということのようだ。スコープ名やログレベルの指定の仕方は <a href="https://github.com/oracle/graal/blob/master/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/debug/doc-files/DumpHelp.txt" target="_blank">Dump help message</a> に詳しく記載されている。

ということで実際にオプションを指定して native-image コマンドを実行してみたが、なんと何も表示されない。どういうことかと思いヘルプを漁ってみたところ以下のオプションを発見した。

```
$ native-image --expert-options-all
  ...
  -H:Log=...                                   Pattern for specifying scopes in which logging is enabled. See the Dump option for the pattern syntax. Default: None
  ...
```

今度はこちらを指定して実行してみたところ、無事にログが出力された。なお、`-H:Log` は experimental なオプションらしく、将来的には `-H:+UnlockExperimentalVMOptions` が必要になる旨の警告が出ていた。

```
$ native-image HelloWorld -H:Log=CreateImage
Warning: The option '-H:Log=CreateImage' is experimental and must be enabled via '-H:+UnlockExperimentalVMOptions' in the future.
Warning: Please re-evaluate whether any experimental option is required, and either remove or unlock it. The build output lists all active experimental options, including where they come from and possible alternatives. If you think an experimental option should be considered as stable, please file an issue.
...
[Use -Djdk.graal.LogFile=<path> to redirect Graal log output to a file.]
[thread:1] scope: main
    [thread:1] scope: main.CreateImage.NativeImage.build
    TextImpl.writeTextSection
    NativeImageHeap.writeHeap:
...
```

上記の出力を見てもらうとわかるように、ファイルに出力するには `-Djdk.graal.LogFile` オプションを使ってねと丁寧に書いてくれている。使ってみたところちゃんとファイルにログが出力された。また、今回例として `CreateImage` というスコープを指定したのだけれど、出力内容を見ると一番上のスコープは `main` ということも見てとれる。ということで `-H:Log=main` と指定してみたところとんでもない量 (GB 単位) のログが出力されたので軽い気持ちでやってみるのは避けたほうがよさそう。

## ちょっとした実験

`logAndIndent` メソッドや `scope` メソッドを使った場合にインデントがどうなるかを軽く実験してみたので残しておく。GraalVM のコード内の適当な箇所に以下を追記してビルド、実行した。

```java
try (DebugContext.Scope s1 = debug.scope("Hoge", new DebugDumpScope("Hoge"))) {
    debug.log("0");
    try (DebugContext.Scope s2 = debug.scope("Fuga", new DebugDumpScope("Fuga"))) {
        debug.log("1");
    }
    try (Indent i1 = debug.logAndIndent("2")) {
        debug.log("3");
        try (Indent i2 = debug.logAndIndent("4")) {
            debug.log("5");
        }
        try (DebugContext.Scope s3 = debug.scope("Piyo", new DebugDumpScope("Piyo"))) {
            debug.log("6");
        }
    }
    debug.log("7");
} catch (Throwable e) {
    throw debug.handle(e);
}
```

結果はこちら。インデントやスコープが変わるタイミングでスレッド、スコープの情報も併せて出力されるらしい。

```
$ native-image HelloWorld -H:Log=Hoge
...
[thread:1] scope: main
  [thread:1] scope: main.Hoge
  0
    [thread:1] scope: main.Hoge.Fuga
    1
  2
    [thread:1] scope: main.Hoge
    3
    4
      [thread:1] scope: main.Hoge
      5
      [thread:1] scope: main.Hoge.Piyo
      6
  7
...
```
