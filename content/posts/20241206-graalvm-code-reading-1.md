+++
title = 'GraalVM Native Image のソースコードを雑に読んだ (1)'
date = 2024-12-06T08:00:06+09:00
slug = 'f68be20f03f370e97cd6f1d81794dcdf'
tags = ['GraalVM']
showtoc = true
+++
この記事は <a href="https://qiita.com/advent-calendar/2024/java" target="_blank">Java Advent Calendar 2024</a> の 6 日目の記事です。

ここ最近 GraalVM の Native Image 関連のソースコードを雑に (本当に雑に) 読んでいたので、その内容をまとめておく。個々の内容 (たとえばコンパイル関連の処理など) についてはたいして深掘りしていないので、そのあたりを期待している方々には物足りない内容であろうことをあらかじめ伝えておきます。全体の流れを掴むために上辺だけをざっと眺めてみた、という感じ。

native-image コマンド実行時に出力されるメッセージによると、バイナリ生成のプロセスは全部で8つのステージに分かれている。そのため、8つの項目に分けて内容をまとめていくことにする。

- <a href="https://www.graalvm.org/latest/reference-manual/native-image/overview/BuildOutput/" target="_blank">Native Image Build Output</a>

```
================================================================================
GraalVM Native Image: Generating 'helloworld' (executable)...
================================================================================
[1/8] Initializing...                                            (2.8s @ 0.15GB)
 Java version: 20+34, vendor version: GraalVM CE 20-dev+34.1
 Graal compiler: optimization level: 2, target machine: x86-64-v3
 C compiler: gcc (linux, x86_64, 12.2.0)
 Garbage collector: Serial GC (max heap size: 80% of RAM)
--------------------------------------------------------------------------------
 Build resources:
 - 13.24GB of memory (42.7% of 31.00GB system memory, determined at start)
 - 16 thread(s) (100.0% of 16 available processor(s), determined at start)
[2/8] Performing analysis...  [****]                             (4.5s @ 0.54GB)
    3,163 reachable types   (72.5% of    4,364 total)
    3,801 reachable fields  (50.3% of    7,553 total)
   15,183 reachable methods (45.5% of   33,405 total)
      957 types,    81 fields, and   480 methods registered for reflection
       57 types,    55 fields, and    52 methods registered for JNI access
        4 native libraries: dl, pthread, rt, z
[3/8] Building universe...                                       (0.8s @ 0.99GB)
[4/8] Parsing methods...      [*]                                (0.6s @ 0.75GB)
[5/8] Inlining methods...     [***]                              (0.3s @ 0.32GB)
[6/8] Compiling methods...    [**]                               (3.7s @ 0.60GB)
[7/8] Laying out methods...   [*]                                (0.8s @ 0.83GB)
[8/8] Creating image...       [**]                               (3.1s @ 0.58GB)
   5.32MB (24.22%) for code area:     8,702 compilation units
   7.03MB (32.02%) for image heap:   93,301 objects and 5 resources
   8.96MB (40.83%) for debug info generated in 1.0s
 659.13kB ( 2.93%) for other data
  21.96MB in total
--------------------------------------------------------------------------------
Top 10 origins of code area:            Top 10 object types in image heap:
   4.03MB java.base                        1.14MB byte[] for code metadata
 927.05kB svm.jar (Native Image)         927.31kB java.lang.String
 111.71kB java.logging                   839.68kB byte[] for general heap data
  63.38kB org.graalvm.nativeimage.base   736.91kB java.lang.Class
  47.59kB jdk.proxy1                     713.13kB byte[] for java.lang.String
  35.85kB jdk.proxy3                     272.85kB c.o.s.c.h.DynamicHubCompanion
  27.06kB jdk.internal.vm.ci             250.83kB java.util.HashMap$Node
  23.44kB org.graalvm.sdk                196.52kB java.lang.Object[]
  11.42kB jdk.proxy2                     182.77kB java.lang.String[]
   8.07kB jdk.graal.compiler             154.26kB byte[] for embedded resources
   1.39kB for 2 more packages              1.38MB for 884 more object types
--------------------------------------------------------------------------------
Recommendations:
 HEAP: Set max heap for improved and more predictable memory usage.
 CPU:  Enable more CPU features with '-march=native' for improved performance.
--------------------------------------------------------------------------------
    0.8s (4.6% of total time) in 35 GCs | Peak RSS: 1.93GB | CPU load: 9.61
--------------------------------------------------------------------------------
Build artifacts:
 /home/janedoe/helloworld/helloworld (executable)
 /home/janedoe/helloworld/helloworld.debug (debug_info)
 /home/janedoe/helloworld/sources (debug_info)
================================================================================
Finished generating 'helloworld' in 17.0s.
```

## 環境

- OS: Linux (WSL 2 + Ubuntu 24.04.1)
- GraalVM: 24.2.0-dev (<a href="https://github.com/oracle/graal/tree/320d02ebb8671b9b7035ff86130fafa859ea012f" target="_blank">320d02ebb867</a>)
- JDK: 21.0.2 (<a href="https://github.com/graalvm/labs-openjdk-21/releases/tag/jvmci-23.1-b33" target="_blank">jvmci-23.1-b33</a>)

## 1. Initializing

> In this stage, the Native Image build process is set up and `Features` are initialized.

先ほども貼ったドキュメント <a href="https://www.graalvm.org/latest/reference-manual/native-image/overview/BuildOutput/#build-stages" target="_blank">Native Image Build Output</a> に各ステージの概要が記載されているので、引用させていただく。以降も同様。

とりあえず、メッセージを出力しているのは以下の2箇所。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGeneratorRunner.java#L437" target="_blank">NativeImageGeneratorRunner.java#L437</a> ( `[1/8] Initializing...` まで )

```java
private int buildImage(ImageClassLoader classLoader) {
    ...
    reporter.printStart(imageName, imageKind);
    ...
```

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L1073" target="_blank">NativeImageGenerator.java#L1073</a> ( `(2.8s @ 0.15GB)` 以降 )

```java
protected void setupNativeImage(OptionValues options, Map<Method, CEntryPointData> entryPoints, JavaMainSupport javaMainSupport,
                SubstitutionProcessor harnessSubstitutions, DebugContext debug) {
        ...
        ProgressReporter.singleton().printInitializeEnd(featureHandler.getUserSpecificFeatures(), loader);
    }
}
```

`setupNativeImage` メソッドまでのスタックトレースはこちら。以前の記事 <a href="http://localhost:1313/posts/a29e6d8462910862f392aa7e0bc07af9/" target="_blank">native-image ファイルの正体 - GraalVM</a> で native-image コマンド実行時に `NativeImageGeneratorRunner` クラスが java コマンドにより実行されているのを見たが、ちゃんと同クラスの `main` メソッドからスタックトレースが始まっていることがわかる。ちなみに以降のステージもすべて `doRun` メソッド内で実行される。

```
setupNativeImage:1073, NativeImageGenerator (com.oracle.svm.hosted)
doRun:565, NativeImageGenerator (com.oracle.svm.hosted)
run:533, NativeImageGenerator (com.oracle.svm.hosted)
buildImage:545, NativeImageGeneratorRunner (com.oracle.svm.hosted)
build:732, NativeImageGeneratorRunner (com.oracle.svm.hosted)
start:151, NativeImageGeneratorRunner (com.oracle.svm.hosted)
main:99, NativeImageGeneratorRunner (com.oracle.svm.hosted)
```

Initializing ステージは `setupNativeImage` メソッドの処理が終わるまで、といえそうである。ただ、初っ端から申し訳ないが初期化関連のコードは読んでいても何をしているか掴みづらくあまりまともに見ていないのでこれ以上何も書けることがない。

せめて、ということで冒頭の引用で登場する `Features` について書いておく。こちらは <a href="https://github.com/oracle/graal/tree/master/sdk" target="_blank">GraalVM SDK</a> に含まれる機能であり、API リファレンスも存在する。

- <a href="https://www.graalvm.org/sdk/javadoc/org/graalvm/nativeimage/hosted/Feature.html" target="_blank">Feature (GraalVM SDK Java API Reference)</a>

> Features allow clients to intercept the native image generation and run custom initialization code at various stages. All code within feature classes is executed during native image generation, and never at run time.

自分も全然理解できていないのだけど、上記の文章からバイナリ生成プロセスの合間合間で custom initialization code を仕込める代物、と読み取れる。確かに、以降もことあるごとに Feature 関連のコードが登場する。たとえば以下。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L849" target="_blank">NativeImageGenerator.java#L849</a>

```java
AfterAnalysisAccessImpl postConfig = new AfterAnalysisAccessImpl(featureHandler, loader, bb, debug);
featureHandler.forEachFeature(feature -> feature.afterAnalysis(postConfig));
```

この例では、analysis 処理が終わった後にいろんな Feature がそれぞれの `afterAnalysis` メソッドによりごにょごにょしている、ということだと思われる。ググっていると以下のサイトも見つけたのでより詳しく知りたければこちらも読んでみるとよいかも。(ただしこちらは公式のドキュメントではなさそう)

- <a href="https://build-native-java-apps.cc/developer-guide/feature/" target="_blank">Feature | Build Native Java Apps with GraalVM</a>

ちょっと読んでみたところ、native-image の `--features` オプションでクラス名を指定することで自作の Feature を使うこともできるらしい。おもしろそうなので今度試してみたい。

## 2. Performing analysis

> In this stage, a <a href="https://dl.acm.org/doi/10.1145/3360610" target="_blank">points-to analysis</a> is performed. The progress indicator visualizes the number of analysis iterations. A large number of iterations can indicate problems in the analysis likely caused by misconfiguration or a misbehaving feature.

メッセージ出力箇所はこちら。`ReporterClosable` インスタンスがクローズされるタイミング、つまり try ブロックを抜けるタイミングで分析結果に関するメッセージが出力される。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L816" target="_blank">NativeImageGenerator.java#L816</a>

```java
protected boolean runPointsToAnalysis(String imageName, OptionValues options, DebugContext debug) {
    ...
    try (ReporterClosable c = ProgressReporter.singleton().printAnalysis(bb.getUniverse(), nativeLibraries.getLibraries())) {
    ...
```

このステージは上記の `runPointsToAnalysis` メソッドで完結している。前述したとおりこちらも `doRun` メソッドから呼び出される。

コードを見る前に事前知識としてこのステージで何をしているかについて軽く触れる。このステージでは静的解析を実施しているわけだが、その内容についてドキュメント内で以下の記載がある。

- <a href="https://www.graalvm.org/latest/reference-manual/native-image/basics/#static-analysis" target="_blank">Native Image Basics</a>

> Static analysis is a process that determines which program elements (classes, methods and fields) are used by an application. These elements are also referred to as **reachable code**.

> Only **reachable** elements are included in the final image.

つまり、ここでいう静的解析とはビルド対象のコードの中で到達可能な要素を調べ上げることを指しているらしい。そして、最終的なイメージにはその要素のみが含まれると。

また、<a href="https://gingk1212.github.io/posts/eee1ddb6f0decf0d2823cf27998be3b5/" target="_blank">GraalVM における universe とは</a> でとりあげた `HostedUniverse` の Javadoc には以下の記述がある。

> A static analysis implements {@link BigBang}. Currently, the only analysis in the project is {@link PointsToAnalysis}, but ongoing research projects investigate different kinds of static analysis.

現在は points-to analysis が GraalVM で使用されている唯一の手法だけど、他の静的解析手法も調査中とのこと。points-to analysis はあくまで静的解析手法の一つということになる。points-to analysis については Wikipedia も見つけたけどこれが同じものを指しているかは正直わからない。

- <a href="https://ja.wikipedia.org/wiki/%E3%83%9D%E3%82%A4%E3%83%B3%E3%82%BF%E8%A7%A3%E6%9E%90" target="_blank">ポインタ解析 - Wikipedia</a>
- <a href="https://en.wikipedia.org/wiki/Pointer_analysis" target="_blank">Pointer analysis - Wikipedia</a>

あとは、冒頭の引用文でリンクされている論文 (まったく読んでない) や、以下のようなドキュメントもあるので適宜参照されたし。

- <a href="https://www.graalvm.org/latest/reference-manual/native-image/debugging-and-diagnostics/StaticAnalysisReports/" target="_blank">Points-to Analysis Reports</a>

事前知識はこれくらいにして、そろそろコードのほうに戻る。やはり気になるのはどこで points-to analysis が実行されているのか。`runPointsToAnalysis` メソッドの中で明らかに分析を実行していそうな箇所がこちら。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.svm.hosted/src/com/oracle/svm/hosted/NativeImageGenerator.java#L821" target="_blank">NativeImageGenerator.java#L821</a>

```java
protected boolean runPointsToAnalysis(String imageName, OptionValues options, DebugContext debug) {
    ...
    bb.runAnalysis(debug, (universe) -> {
        try (StopTimer t2 = TimerCollection.createTimerAndStart(TimerCollection.Registry.FEATURES)) {
            bb.getHostVM().notifyClassReachabilityListener(universe, config);
            featureHandler.forEachFeature(feature -> feature.duringAnalysis(config));
        }
        return !config.getAndResetRequireAnalysisIteration();
    });
    ...
```

`runAnalysis` メソッドは `BigBang` インターフェースで定義されているメソッド。`BigBang` は先ほどの引用にも登場していたが、Javadoc によると Central static analysis interface らしい。デバッグ実行してみると、実際に呼ばれているのは `AbstractAnalysisEngine` の `runAnalysis` メソッドだった。 (※デバッグ実行のやり方 → <a href="https://gingk1212.github.io/posts/d671ee72f6e63699662a7d0467cc9f44/" target="_blank">GraalVM のバイナリ生成プロセスをデバッグ実行する</a>)

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/AbstractAnalysisEngine.java#L153" target="_blank">AbstractAnalysisEngine.java#L153</a>

```java
public void runAnalysis(DebugContext debugContext, Function<AnalysisUniverse, Boolean> analysisEndCondition) throws InterruptedException {
    int numIterations = 0;
    while (true) {
        try (Indent indent2 = debugContext.logAndIndent("new analysis iteration")) {
            /*
             * Do the analysis (which itself is done in a similar iterative process)
             */
            boolean analysisChanged = finish();
            ...
```

冒頭の引用にもある通りここでのイテレーション回数が `[****]` という形で出力される。コメントによると実際に analysis を実行しているのは `finish` メソッドのようなので、さらに中身を見てみる。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/PointsToAnalysis.java#L623" target="_blank">PointsToAnalysis.java#L623</a>

```java
/**
 * Performs the analysis.
 *
 * @return Returns true if any changes are made, i.e. if any type flows are updated
 */
@SuppressWarnings("try")
@Override
public boolean finish() throws InterruptedException {
    try (Indent indent = debug.logAndIndent("starting analysis in BigBang.finish")) {
        boolean didSomeWork = false;
        do {
            didSomeWork |= doTypeflow();
            assert executor.getPostedOperations() == 0 : executor.getPostedOperations();
            universe.runAtFixedPoint();
        } while (executor.getPostedOperations() > 0);
        return didSomeWork;
    }
}
```

あやしそうなのは `doTypeflow` メソッド。`finish` メソッドのすぐ下に定義されている。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/PointsToAnalysis.java#L636" target="_blank">PointsToAnalysis.java#L636</a>

```java
@SuppressWarnings("try")
public boolean doTypeflow() throws InterruptedException {
    boolean didSomeWork;
    try (StopTimer ignored = typeFlowTimer.start()) {
        executor.start();
        executor.complete();
        didSomeWork = (executor.getPostedOperations() > 0);
        executor.shutdown();
    }
    /* Initialize for the next iteration. */
    executor.init(timing);
    return didSomeWork;
}
```

`executor` というものを使って処理をしているように見える。`executor` は `CompletionExecutor` クラスのインスタンス。`CompletionExecutor` の `start` , `complete` メソッドをのぞいてみる。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/util/CompletionExecutor.java#L183" target="_blank">CompletionExecutor.java#L183</a>

```java
public void start() {
    assert state.get() == State.BEFORE_START : state.get();

    setState(State.STARTED);
    postedBeforeStart.forEach(this::execute);
    postedBeforeStart = null;
}
```

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/util/CompletionExecutor.java#L152" target="_blank">CompletionExecutor.java#L152</a> (上記 `this::execute` の先で呼ばれているメソッド)

```java
private void executeService(DebugContextRunnable command) {
    ForkJoinPool.commonPool().execute(() -> executeCommand(command));
}
```

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/util/CompletionExecutor.java#L208" target="_blank">CompletionExecutor.java#L208</a>

```java
public long complete() throws InterruptedException {
    ...
    boolean quiescent = ForkJoinPool.commonPool().awaitQuiescence(100, TimeUnit.MILLISECONDS);
    ...
```

つまり、`start` メソッドで複数のタスクを起動して、`complete` メソッドで終了するまで待機しているのだろう。ということは、分析タスクはどこかで事前に `executor` に登録されていると思われる。デバッグ実行して実際にタスクが実行されている箇所まで進めてみると、たとえば以下の箇所にたどり着いた。ここでは `BigBang` インターフェースの `postTask` メソッドにより実行すべきタスクを登録しているようだ。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/meta/AnalysisType.java#L649" target="_blank">AnalysisType.java#L649</a>

```java
@Override
protected void onReachable(Object reason) {
    ...
    universe.getBigbang().postTask(unused -> forAllSuperTypes(this::prepareMethodImplementations, false));
    ...
}
```

なんとなく大まかな流れはわかったので、最後に分析結果を出力して確認してみる。<a href="https://gingk1212.github.io/posts/eee1ddb6f0decf0d2823cf27998be3b5/" target="_blank">GraalVM における universe とは</a> に書いた通り、静的解析が取り扱う型、メソッド、フィールドは "analysis universe" と呼ばれ、それぞれ `AnalysisType`, `AnalysisMethod`, `AnalysisField` というクラスが対応する。たとえば `AnalysisMethod` には `isReachable` というメソッドがあるので、これを呼んでちゃんと結果が返ってくることを確認してみる。ビルド対象は以下のコードとした。

- SampleApp.java

```java
public class SampleApp {
  public static void main(String[] args) {
    System.out.println("Hello, World!");
    hoge();
  }

  private static void hoge() {
    System.out.println("Hello from hoge!");
  }
}
```

`hoge` メソッドに到達可能かどうかを出力する以下のコードを `runAnalysis` メソッド呼び出しの直後に追記してビルド。 (※ビルド方法 → <a href="https://gingk1212.github.io/posts/34f2d9d91c05126ef05570b0993b9534/" target="_blank">GraalVM をビルドする</a>)

```java
AnalysisMethod hoge =
    aUniverse.getMethods().stream()
        .filter(analysisMethod -> analysisMethod.getName().equals("hoge"))
        .findFirst()
        .get();
System.out.println(hoge.isReachable());
```

SampleApp を指定して native-image を実行してみたところ、想定通り `true` が出力された。なお、`aUniverse` は `NativeImageGenerator` クラスが持つ `AnalysisUniverse` 型のフィールドであり、以下のように型、メソッド、フィールドを保持している。`runAnalysis` メソッドが呼び出される前は `methods` の中にそもそも `hoge` が含まれていなかった。

- <a href="https://github.com/oracle/graal/blob/320d02ebb8671b9b7035ff86130fafa859ea012f/substratevm/src/com.oracle.graal.pointsto/src/com/oracle/graal/pointsto/meta/AnalysisUniverse.java#L90" target="_blank">AnalysisUniverse.java#L90</a>

```java
private final ConcurrentMap<ResolvedJavaType, Object> types = new ConcurrentHashMap<>(ESTIMATED_NUMBER_OF_TYPES);
private final ConcurrentMap<ResolvedJavaField, AnalysisField> fields = new ConcurrentHashMap<>(ESTIMATED_FIELDS_PER_TYPE * ESTIMATED_NUMBER_OF_TYPES);
private final ConcurrentMap<ResolvedJavaMethod, AnalysisMethod> methods = new ConcurrentHashMap<>(ESTIMATED_METHODS_PER_TYPE * ESTIMATED_NUMBER_OF_TYPES);
```

## おわりに

思ったより長くなりそうなので今回はここまでにしておく。読んでもらうとわかる通り推測も大いに含まれる内容となっておりますのでその点についてはご留意くださいませ。
