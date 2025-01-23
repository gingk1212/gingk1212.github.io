+++
title = 'GraalVM Natvie Image でグラフをダンプする'
date = 2025-01-23T09:00:00+09:00
slug = '6c99c61e30ed6e7ef16794d9da655e99'
tags = ['GraalVM']
showtoc = true
+++
GraalVM の native-image によるバイナリ生成時にグラフをダンプする方法を調べたのでまとめておく。参考にしたサイトは以下。

- <a href="https://www.graalvm.org/latest/reference-manual/native-image/overview/Options/#graph-dumping" target="_blank">Command-line Options - Graph Dumping</a>
    - Native Image 機能使用時にグラフをダンプするためのオプションについて (公式ドキュメント)
- <a href="https://www.graalvm.org/latest/tools/igv/" target="_blank">Ideal Graph Visualizer</a>
    - ダンプしたグラフを表示したり解析したりするための GUI ツール Ideal Graph Visualizer (IGV) についての公式ドキュメント
- <a href="https://logico-jp.dev/2020/06/14/understanding-basic-graal-graphs/" target="_blank">Understanding Basic Graal Graphs &#8211; Logico Inside</a>
    - Shopify の中の人の記事の日本語訳
    - グラフの見方について詳しく説明されている

使用するサンプルコードはこちら。グラフの内容には触れないのでなんでもよいのだけれど、<a href="https://github.com/oracle/graal/issues/9320" target="_blank">oracle/graal#9320</a> の issue について調べている過程でグラフを見てみたくなってダンプ方法を調べたので、こちらの issue の再現コードを使用する。

- zzz.java

```java
public class zzz {
  public static void main(String args[]) {
    hoge();
  }

  public static void hoge() {
    int[] pos = new int[11];
    int mov = 1;
    for (; ; ) {
      for (int i = pos.length - 1; i > 0; i--) {
        pos[i] = pos[i - 1];
      }
      pos[0] += mov;
    }
  }
}
```

## グラフのダンプ

結論、ダンプコマンドは以下の通り。`zzz` クラスの `hoge` メソッドをダンプ対象として指定している。

```
$ native-image zzz -H:Dump= -H:MethodFilter=zzz.hoge
```

コマンド実行後、graal_dumps というディレクトリが作成され、2つの bgv ファイルが生成される。

```
$ tree graal_dumps/
graal_dumps/
└── 2025.01.23.01.16.40.367
    ├── PointsToAnalysisMethod:18438[zzz.hoge()void].bgv
    └── SubstrateHostedCompilation-3245[zzz.hoge()void].bgv

2 directories, 2 files
```

## IGV でグラフを表示する

次に、ダンプしたグラフを表示する。まずは mx にパスが通っている場所で以下のコマンドを実行する。

```java
$ mx igv
```

すると、冒頭でも触れた Ideal Graph Visualizer (IGV) が起動するので、bgv ファイルを開いていく。

![20250123-graalvm-graph-dump-01.png](../image/20250123-graalvm-graph-dump-01.png)

2つ同時に選択。

![20250123-graalvm-graph-dump-02.png](../image/20250123-graalvm-graph-dump-02.png)

以下の通り、Outline の領域に読み込んだ内容が表示される。

![20250123-graalvm-graph-dump-03.png](../image/20250123-graalvm-graph-dump-03.png)

試しに一つ開いてみた様子が以下。無事グラフが表示された。

![20250123-graalvm-graph-dump-04.png](../image/20250123-graalvm-graph-dump-04.png)

ソースコードを紐づけることもできる。Source Collections という領域で右クリックし、Add source root を選択。 (Source Group が何かはわかりません・・)

![20250123-graalvm-graph-dump-05.png](../image/20250123-graalvm-graph-dump-05.png)

ソースコードが置いてあるフォルダを選択すると以下の通りソースが読み込まれる。

![20250123-graalvm-graph-dump-06.png](../image/20250123-graalvm-graph-dump-06.png)

すると、グラフ上のノードを選択すると対応するソースコードの行がハイライトされるようになる。

![20250123-graalvm-graph-dump-07.png](../image/20250123-graalvm-graph-dump-07.png)

## グラフがダンプされるタイミング

最後に、グラフがダンプされるタイミングについて触れておく。先ほど貼った画像を再掲。一つの bgv ファイルにつき一つのグラフ、というわけではないらしい。

![20250123-graalvm-graph-dump-03.png](../image/20250123-graalvm-graph-dump-03.png)

名前からして、Performing analysis ステージと Compiling methods ステージ中にダンプされたように見える。なお、これらのステージで何をしているかについては以下の記事で触れている。

- [GraalVM Native Image のソースコードを雑に読んだ (1)](/posts/f68be20f03f370e97cd6f1d81794dcdf/)
- [GraalVM Native Image のソースコードを雑に読んだ (3)](/posts/49278a58f529469d8f52dfcb36cf3e3e/)

実際にコードを確認してみる。まずは `InlineBeforeAnalysis` クラスを見てみると、グラフをダンプしている箇所を発見。

- <a href="https://github.com/oracle/graal/blob/f6e524983ab6bdc1ccce9693cf9b01eadf8c737f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/phases/InlineBeforeAnalysis.java#L74" target="_blank">InlineBeforeAnalysis.java#L74</a>

```java
public static StructuredGraph decodeGraph(BigBang bb, AnalysisMethod method, AnalysisParsedGraph analysisParsedGraph) {
    ...

    StructuredGraph result = new StructuredGraph.Builder(bb.getOptions(), debug, bb.getHostVM().allowAssumptions(method))
                    .method(method)
                    .trackNodeSourcePosition(analysisParsedGraph.getEncodedGraph().trackNodeSourcePosition())
                    .recordInlinedMethods(analysisParsedGraph.getEncodedGraph().isRecordingInlinedMethods())
                    .build();

    ...
    debug.dump(DebugContext.BASIC_LEVEL, result, "InlineBeforeAnalysis after decode");
    ...
```

こちらは案の定、Performing analysis ステージ内で実行されていた。

次に、もう一方のダンプがいつ実行されているかを確認。`Before phase` などの文字列でソースコードを検索した結果、一つ目のグラフは以下の箇所でダンプされていた。

- <a href="https://github.com/oracle/graal/blob/master/compiler/src/jdk.graal.compiler/src/jdk/graal/compiler/phases/BasePhase.java#L363" target="_blank">BasePhase.java#L363</a>

```java
private boolean dumpBefore(final StructuredGraph graph, final C context, boolean isTopLevel, boolean applied) {
    ...
    debug.dump(DebugContext.BASIC_LEVEL, graph, "Before phase %s%s", getName(), tag);
    ...
```

こちらは以前の記事でも見た `emitFrontEnd` メソッド内の `suites.getHighTier().apply(graph, highTierContext)` という処理の中で呼ばれている。また、以降の処理中で `After xxx tier` という名前でグラフをダンプしていることもわかる。

```java
public static void emitFrontEnd(Providers providers, TargetProvider target, StructuredGraph graph, PhaseSuite<HighTierContext> graphBuilderSuite, OptimisticOptimizations optimisticOpts,
                ProfilingInfo profilingInfo, Suites suites) {
    ...
    suites.getHighTier().apply(graph, highTierContext);
    graph.maybeCompress();
    debug.dump(DebugContext.BASIC_LEVEL, graph, "After high tier");

    MidTierContext midTierContext = new MidTierContext(providers, target, optimisticOpts, profilingInfo);
    suites.getMidTier().apply(graph, midTierContext);
    graph.maybeCompress();
    debug.dump(DebugContext.BASIC_LEVEL, graph, "After mid tier");

    LowTierContext lowTierContext = new LowTierContext(providers, target);
    suites.getLowTier().apply(graph, lowTierContext);
    debug.dump(DebugContext.BASIC_LEVEL, graph, "After low tier");
    ...
```

つまり、こちらも案の定 Compiling methods ステージ内でダンプが実行されているということになる。
