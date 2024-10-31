+++
title = 'GraalVM における universe とは'
date = 2024-10-31T09:51:52+09:00
slug = 'eee1ddb6f0decf0d2823cf27998be3b5'
tags = ['GraalVM']
showtoc = true
+++
GraalVM の Native Image に関するソースコードを読んでいるとよく目にする universe という単語。以下の通りバイナリ生成時のメッセージにも登場しているのだけれど、最初は何のことなのかまったくわからなかった。

```
$ native-image HelloWorld
========================================================================================================================
GraalVM Native Image: Generating 'helloworld' (executable)...
========================================================================================================================
...
[3/8] Building universe...                                                                               (0.9s @ 0.42GB)
...
========================================================================================================================
Finished generating 'helloworld' in 12.3s.
```

あまりにも頻出するのでさすがに意味を知っておきたいと思い公式ドキュメントやリポジトリ内の .md ファイルを探してみたのだけれどこれといったものは見つからず。あるいはコンパイラ関連の用語かと思ってググってみるもこちらも何も引っかからず。唯一見つかったのはバイナリ生成時のメッセージについて説明しているドキュメント <a href="https://www.graalvm.org/latest/reference-manual/native-image/overview/BuildOutput/#building-universe" target="_blank">Native Image Build Output</a> 内の以下の説明。これだけは何のことやら。

> In this stage, a universe with all types, fields, and methods is built, which is then used to create the native binary.

ということでひとまず置いておいてコードを読んでいると、なんと `HostedUniverse` というクラスの Javadoc に詳細な説明が書いてあるのを見つけた。けっこう重要な話だと思うのでせめて .md ファイルにあってもいいのになあと思ったり。

- <a href="https://github.com/oracle/graal/blob/54826272bd922cb3222de4decd5fc8f69621c4df/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/meta/HostedUniverse.java#L92" target="_blank">HostedUniverse.java#L92</a>

## universe とは

冒頭を読むと universe とは何かがなんとなくわかると思うので、日本語訳も残しておく。(意訳含む)

---

native image generator は、型 (`ResolvedJavaType`)、メソッド (`ResolvedJavaMethod`)、フィールド (`ResolvedJavaField`) に関する JVMCI インターフェースの実装を複数のレイヤーで使用します。このドキュメントでは、型、メソッド、フィールドを指して "element" という用語を使用します。 ある特定のレイヤーのすべての elements を "universe" と呼びます。

使用されているレイヤーは4つあります:

- The "host VM universe": クラスファイルからパースされた elements のオリジナルソース
- The "substitution layer": クラスファイルを変更することなく、クラスファイルから提供される elements の一部を変更するためのレイヤー
- The "analysis universe": 静的解析が操作する elements
- The "hosted universe": AOT コンパイルが操作する elements

---

> このドキュメントでは、型、メソッド、フィールドを指して "element" という用語を使用します。 ある特定のレイヤーのすべての elements を "universe" と呼びます。

universe についてはこちらに記載の通り。"すべての" だから宇宙的な意味で "universe" ということなのだろうか。このあたりの英語のニュアンスはよくわからない。

ちなみに `ResolvedJavaType`, `ResolvedJavaMethod`, `ResolvedJavaField` は JVMCI に存在するインターフェース。以下のパスにファイルが置いてある。

- <a href="https://github.com/graalvm/labs-openjdk-21/tree/jvmci-23.1-b33/src/jdk.internal.vm.ci/share/classes/jdk/vm/ci/meta" target="_blank">src/jdk.internal.vm.ci/share/classes/jdk/vm/ci/meta</a>

また、冒頭に載せた `[3/8] Building universe...` というステージでは4つのレイヤーのうちの "hosted universe" を生成している。ソースコードとしてはこのあたりが該当。

- <a href="https://github.com/oracle/graal/blob/a7cae0179c3516c6073179f08a48a151c96ad0d7/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L616" target="_blank">NativeImageGenerator.java#L616</a>

```java
protected void doRun(Map<Method, CEntryPointData> entryPoints, JavaMainSupport javaMainSupport, String imageName, NativeImageKind k, SubstitutionProcessor harnessSubstitutions) {
    ...
    new UniverseBuilder(aUniverse, bb.getMetaAccess(), hUniverse, hMetaAccess, HostedConfiguration.instance().createStrengthenGraphs(bb, hUniverse),
                bb.getUnsupportedFeatures()).build(debug);
    ...
```

## おわりに

残りの部分についても一通り読みはしたが、文章だけではよく理解できない部分も多々あったのであとは実装を見つつ必要に応じて参照しようと思う。興味のある方はぜひドキュメントを読んでみてください。
